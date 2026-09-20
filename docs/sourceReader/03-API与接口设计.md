# 03-API 与接口设计

本篇按使用面组织 rig 的对外 API，全部为当前代码的真实签名（文件:行号可定位）。

---

## 🧠 接口总览

| 抽象层 | 主要入口 | 路径 |
|--------|----------|------|
| 客户端/模型 | `Client::new/from_env/builder`、`completion_model`、`embedding_model` | `crates/rig-core/src/client/` |
| 补全 | `CompletionModel::{completion,stream}`、`completion_request(...)` builder | `crates/rig-core/src/completion/` |
| Agent | `client.agent(model)` → `AgentBuilder` → `Agent::prompt/run/stream/chat/resume` | `crates/rig-agent/src/agent/` |
| 工具 | `#[rig_tool]`、`impl Tool`、`ToolSet` | `crates/rig-core/src/tool/`、`crates/rig-agent/src/tool/` |
| 向量库 | `VectorStoreIndex::{top_n,top_n_ids}` + 各伴侣 crate builder | `crates/rig-core/src/vector_store/` |

---

## 1️⃣ 客户端构造（三段式）

以 OpenAI 为例（其余 provider 同构）：

```rust
use rig::prelude::*;   // 打开 CompletionClient + DefaultTransportClient 等便捷 trait

// ① 一参构造（reqwest 传输，Bearer key）
let client = openai::Client::new("sk-...")?;

// ② 环境变量（OPENAI_API_KEY，可选 OPENAI_BASE_URL）
let client = openai::Client::from_env()?;

// ③ 显式 builder（无传输状态开始）
let client = openai::Client::builder()
    .api_key("sk-...")
    // .base_url("https://...")        // 覆盖 BASE_URL
    // .http_headers(headers)          // 追加默认头
    .build()?;                          // reqwest feature 下 build 可缺传输
```

底层：`new`/`from_env` 来自 `rig-reqwest::client::DefaultTransportClient`（门面 prelude 重导出，`src/lib.rs:147`）；不带 reqwest feature 时必须 `builder().http_client(h)` 显式给传输（`ClientBuilder::http_client`，`client/mod.rs:705`）。无凭据 provider（ollama、llamacpp 等）用 `Nothing` key。

```rust
// 环境变量辅助（provider 实现内部用）
Client::from_env_api_key("OPENAI_API_KEY", Some("OPENAI_BASE_URL"), http)?;  // client/mod.rs:398
```

### 能力方法（能力 trait 由 provider 决定有没有）

```rust
let model: openai::CompletionModel<_> = client.completion_model(openai::GPT_5_2);  // "gpt-5.2"
let embedding_model = client.embedding_model("text-embedding-3-small");
let lister = client.list_models();
client.verify().await?;    // 打 VERIFY_PATH 验凭据
```

模型常量示例（`providers/openai/completion/mod.rs:41` 起）：`GPT_5_6`、`GPT_5_6_SOL`、`GPT_5_5`、`GPT_5_2`、`GPT_5_1`、`GPT_5`…

---

## 2️⃣ 补全请求与调用

`CompletionRequest`（`completion/request.rs:805`）字段：`model`、`chat_history: Vec<Message>`（最后一条是 prompt，"至少一条"是规则而非类型保证，构造边界校验）、`documents`、`tools: Vec<ToolDefinition>`、`temperature`、`max_tokens`、`tool_choice`、`additional_params: serde_json::Value`、`output_schema: schemars::Schema`、`record_telemetry_content`。

消息模型（`completion/message.rs:20`）：

```rust
pub enum Message {
    System { content: String },
    User { content: Vec<UserContent> },        // Text/ToolResult/Image/Audio/Video/Document
    Assistant { id: Option<String>, content: Vec<AssistantContent> },  // Text/ToolCall/Reasoning/Image
}
```

直接调用模型（不经 agent）：

```rust
let request = model.completion_request("What is Rig?").build();   // CompletionRequestBuilder
let response = model.completion(request).await?;      // CompletionResponse
let stream = model.stream(request).await?;            // StreamingCompletionResponse (StreamEvent 流)
```

流事件是**块模型**（`streaming/event.rs:35`）：`BlockStart{id,kind}` → `BlockDelta{id,delta}`* → `BlockEnd{id,end,…}`，块的 `BlockId` 贯穿流的生命周期；文本/推理/工具调用参数都是 delta，块结束携带终结记录。

---

## 3️⃣ Agent：构造与运行

### 构造（builder）

```rust
let agent = client
    .agent(openai::GPT_5_2)                       // AgentClientExt::agent（client.rs:26）
    .name("helper")                               // builder.rs:163
    .preamble("You are a helpful assistant.")
    .context("静态上下文文档")                     // 静态文档
    .dynamic_context(3, vector_index)             // 每轮检索（Retrieves 效果）
    .tool(weather_tool)                           // 进入 WithBuilderTools 状态
    .dynamic_tool(DynamicTool { .. })             // 动态定义的工具
    .temperature(0.8)
    .max_tokens(1024)
    .tool_choice(ToolChoice::Required)
    .memory(memory_backend)                       // 会话记忆（load/append 经 bus）
    .conversation(ConversationId::new("c1"))
    .default_max_turns(10)                        // 总模型调用预算
    .add_hook(my_hook)                            // builder.rs:411
    .model_route("fast", cheap_model)             // 多模型路由（on_model_select 可切）
    .record_effects()                             // 记 effect 日志
    .build();                                     // 直接返回 Agent（builder.rs:659）
```

### 四个运行入口（`agent/completion.rs` / `agent/runner.rs`）

```rust
// 一次性提问，聚合响应
let response: PromptResponse = agent
    .prompt("What is 2+2?")            // completion.rs:732 → AgentRunner
    .run()                             // runner.rs:535 → impl Future<Output=Result<PromptResponse,PromptError>>
    .await?;

// 多轮流式：run 与 stream 共享同一循环，事件一致
let stream: StreamingResult = agent.prompt("...").stream().await?;  // streaming.rs:305
// StreamingResult = Pin<Box<dyn Stream<Item = Result<MultiTurnStreamItem, StreamingError>>>>

// 拆分驱动：返回 (驱动 future, 事件 feed)
let (fut, events: RunEvents) = agent.prompt("...").run_channel();   // streaming.rs:485

// 会话式：自带历史
agent.chat("hi", &mut history).await?;                              // completion.rs:791

// 断点续跑：持久化过 AgentRun
agent.resume(serialized_run);                                       // completion.rs:776
```

`PromptResponse`（`run/response.rs:119`）：`output: String`（最终轮聚合文本）、`usage: Usage`（全程聚合）、`agent_history: Vec<CompletionCall>`、`messages: Option<Vec<Message>>`、`finish_reason` 等。每个 `CompletionCall`（`run/response.rs:11`）按调用记录 `call_index/usage/message_id/response_id/provider_request_id/finish_reason/raw`（**provider 原始响应**，#2366）。

### 结构化输出

```rust
// 类型化提取（agent 上的便捷）
let res: T = agent.prompt_typed::<T>("...").run().await?.extract()?;  // TypedRun<T>

// 经典提取器
let extractor = client.extractor::<Sentiment>(&openai::GPT_5_2);   // AgentClientExt::extractor
// OutputMode（run/output.rs:28）决定落点：
//   Auto（默认，有 schema+工具时用 Tool）、Tool（合成输出工具）、Native（provider 原生约束）、Prompted（注入提示词）
```

---

## 4️⃣ 工具定义

### `#[rig_tool]` 属性宏（crates/rig-derive/src/lib.rs:171）

```rust
#[rig_tool(
    description = "Get the current weather",
    params(city = "The city to query")      // 参数文档注入 schema
)]
async fn get_weather(city: String) -> Result<String, MyError> { ... }
```

宏生成一个实现 `PortableTool` 的零数据结构体（自动 blanket 到 `Tool`）。

### 手写上下文工具

```rust
impl Tool for WeatherTool {
    const NAME: &'static str = "get_weather";
    type Args = WeatherArgs;        // serde 反序列化
    type Output = String;          // 任意 IntoToolOutput；ToolResultContent 保留富内容
    type Error = MyError;

    fn description(&self) -> String { "Get weather".into() }
    fn parameters(&self) -> serde_json::Value { schema_json() }

    async fn call(&self, ctx: &mut ToolContext, args: Self::Args) -> Result<String, MyError> {
        // ctx: ContextValue 共享表（agent 侧注入 API key、HTTP 客户端等）
        ...
    }
}
```

注册：builder 的 `.tool()`/`.dynamic_tool()`/`.dynamic_tools()`，或运行时经 `ToolServer`（远端源/MCP 同步）。执行侧：`ToolCatalog` 每轮过滤 → `execute_tool` 按名分发。

---

## 5️⃣ 向量检索

```rust
// 建索引（任意伴侣 crate，以 in-memory 为例）
let mut store = InMemoryVectorStore::from_documents(vec![(doc, embedding)]);  // vector_store/in_memory_store.rs:83

// 查询（VectorSearchRequest 携带查询文本与样本数，存储内部负责嵌入）
let hits: Vec<(f64, String, Doc)> = store
    .top_n(VectorSearchRequest::builder().query("...").samples(5).build())
    .await?;
```

伴侣 crate 各有自己的 builder（`rig_qdrant::Qdrant::builder()...`、`rig_lancedb::...`），统一实现 `VectorStoreIndex`。接入 agent：`.dynamic_context(samples, index)`（检索为 Retrieve 效果，可被 hook 门控）或独立检索流程。

---

## 6️⃣ 多轮流式事件面

`stream()` 与 `run_channel()` 产出 `MultiTurnStreamItem`（`agent/streaming.rs:41`）：

- `StreamAssistantItem(StreamEvent)` — provider 原始块事件；
- `ToolCall { .. }` — 模型轮提交时报告的模型工具调用；
- `ToolExecutionCommitted { .. }` — 工具执行生命周期；
- `StreamUserItem(StreamedUserContent)` — 工具结果等以用户内容回填；
- `CompletionCall(CompletionCall)` — 每次补全的调用详情（含 raw）。

`RunEvents::try_next()` 非阻塞取一件，`is_done()` 判断结束（`agent/streaming.rs:428`）。

---

## 📁 文件路径映射表

| 功能 | 路径 |
|------|------|
| Provider/Has*/Client/ClientBuilder | `crates/rig-core/src/client/mod.rs` |
| CompletionModel/Request/Error | `crates/rig-core/src/completion/request.rs` |
| Message/内容部件 | `crates/rig-core/src/completion/message.rs` |
| 流块模型 | `crates/rig-core/src/streaming/event.rs` |
| AgentBuilder | `crates/rig-agent/src/agent/builder.rs` |
| Agent 入口（prompt/resume/chat） | `crates/rig-agent/src/agent/completion.rs` |
| AgentRunner（run/run_channel） | `crates/rig-agent/src/agent/runner.rs`、`agent/streaming.rs` |
| AgentHook/HookStack | `crates/rig-agent/src/agent/hook.rs` |
| AgentRun 状态机 | `crates/rig-agent/src/run/mod.rs` |
| Extractor/OutputMode | `crates/rig-agent/src/extractor.rs`、`src/run/output.rs` |
| Tool/PortableTool | `crates/rig-core/src/tool/{contextual,portable}.rs` |
| `#[rig_tool]` | `crates/rig-derive/src/lib.rs:171` |
| ToolSet/Catalog/Server | `crates/rig-agent/src/tool/{registry,catalog,server}.rs` |
| effect 词汇 / serve 契约 | `crates/rig-core/src/effect/mod.rs`、`src/serve/handler.rs` |
| VectorStoreIndex | `crates/rig-core/src/vector_store/mod.rs:133` |
| 门面重导出矩阵 | `src/lib.rs` |