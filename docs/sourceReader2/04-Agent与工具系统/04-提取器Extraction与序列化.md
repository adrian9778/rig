# 04-04 · 提取器 Extraction 与可序列化运行状态

> 结构化提取不是独立引擎，而是 Agent 的一组固定配置：`OutputMode::Tool` + 合成 `submit` 工具 + `ToolChoice::Required` + 单轮预算 + 类型化解包。本篇拆 `Extractor<T>`/`TypedRun<T>`，以及 sans-IO 状态机的序列化边界。
> 文件：`crates/rig-agent/src/extractor.rs`、`agent/typed.rs`、`run/mod.rs`。

## 1. 定位

```mermaid
flowchart LR
    EB[ExtractorBuilder&lt;T&gt; :109] -->|build| E[Extractor&lt;T&gt; :53]
    E -->|extract(text) :100| TR[TypedRun&lt;T&gt; typed.rs:103]
    TR -->|history/retries/using_model 再配置| AW[.await]
    AW --> TPR[TypedPromptResponse&lt;T&gt;]
    subgraph 固定配方
      D1[preamble: 必须调 submit]
      D2[output_schema::&lt;T&gt;]
      D3[ToolChoice::Required]
      D4[OutputMode::Tool]
      D5[max_turns = 1]
    end
    EB -.固定.-> D1 & D2 & D3 & D4
    E -.extract 时.-> D5
```

## 2. Why：提取为什么建模成"输出工具"

1. **Native 结构化输出会压制工具调用**：部分 provider（如 Ollama）的 `response_format` 约束每一轮，模型无法先调用检索/计算工具再产出结构化结果。`OutputMode::Tool` 只把 schema 变成一个名为 `submit` 的合成函数，模型可以先自由用别的工具（见 `run/output.rs:28` 注释与 #1928）。
2. **类型只在边界出现一次**：作者定义 `T: JsonSchema + DeserializeOwned`，schemars 生成 schema、运行时解包成 `T`，中间全部是 `serde_json::Value` 与同一个工具协议——没有第二条"结构化响应"管线。
3. **失败可计费、可重试**：模型没调 `submit` 是一次空响应而非错误；在重试预算内**从头重跑**（usage 跨尝试累计，包括收到了计费响应但解包失败的尝试；补全调用本身报错的尝试不计 usage——extractor.rs:90-96 注释）。

## 3. What：类型与 API

### 3.1 ExtractorBuilder — `extractor.rs:109`

```rust
ExtractorBuilder::<T>::new(model)            // :135
    .from_agent_builder(AgentBuilder::new(model))  // :145，固定配方在此注入
    .append_preamble("…")                    // :167，只能追加，固定 preamble 不可替换
    .context("…") .additional_params(..) .max_tokens(..)
    .tool_choice(..) .add_hook(..)           // forward_agent_builder! 宏原样转发
    .dynamic_context(samples, index)         // 委托 AgentBuilder::dynamic_context
    .retries(n)                              // builder 本地：提取重试预算
    .build() -> Extractor<T>
```

`from_agent_builder`（:145）注入：固定 preamble（"你是结构化提取器…**必须调用 submit**"）、`.output_schema::<T>()`、`ToolChoice::Required`、`OutputMode::Tool`。bound 为 `T: JsonSchema + DeserializeOwned + Serialize + WasmCompatSend + WasmCompatSync + 'static`。

### 3.2 Extractor — `extractor.rs:53`

- `extract(text) -> TypedRun<T>`（:100）：`agent.prompt(text).max_turns(1).output_tool("submit", "Submit the structured data…", false)`，再包成 `TypedRun::output_tool(runner).retries(self.retries)`。
- `with_model_ref(label)`（:63）/`with_model(model)`（:71）：换后续提取用的模型（走 bus 标签或直接注册）。
- `agent()`（:80）：借内部 Agent 做其他配置。

### 3.3 TypedRun 与类型化响应 — `agent/typed.rs`

`TypedRun<T>:103` 暴露与 `AgentRunner` 同形的逐 run 配置（宏生成，typed.rs:119 起）：`tool_context/history/preamble/document(s)/temperature/max_tokens/additional_params/using_model/retries`。终端 `.await` 得 `TypedPromptResponse<T>`；`TypedPromptResponse::new(output, usage):57`、`with_completion_calls:67`、`completion_calls():76`、`requests():81`。

### 3.4 三种结构化形态的选择（回顾 `run/output.rs:28`）

| 需求 | 模式 |
|---|---|
| 提取前要先调工具、要最大兼容性（Extractor 的选择） | `Tool` |
| 要 provider 保证 schema 合规、不需要工具 | `Native` |
| 弱/本地模型，schema 放 prompt，自己解析脏 JSON | `Prompted` |
| 让 Rig 按"有无工具+schema"自动选 | `Auto`（默认） |

## 4. How：一次提取的数据流

```mermaid
sequenceDiagram
    participant U
    participant E as Extractor&lt;T&gt;
    participant A as Agent（Tool 模式）
    participant M as 模型
    U->>E: extract(text).history(..).await
    E->>A: prompt + output_tool(submit, schema=T) + max_turns 1
    A->>M: 补全（工具列表含业务工具 + submit）
    alt 模型调用 submit(args)
        M-->>A: tool call
        A->>A: 不执行 submit；args 按 T 解包
        alt 解包成功
            A-->>U: TypedPromptResponse { output: T, usage, completion_calls }
        else 缺必填/类型错（预算内）
            A->>M: 带纠正反馈重问（消耗 max_output_retries）
        end
    else 终轮无 submit
        A->>A: 视为空提取，预算内从头重跑（usage 累计）
    end
```

要点：submit 是**合成输出工具**（AgentRun 的 `output_tool_name`），模型调它即终局，不进工具执行路径（见 04-01 篇 §3.1）。

## 5. 可序列化运行状态

`run/mod.rs` 顶部注释（:17）明确整个 run 状态是 `Serialize + Deserialize`：

- 全部状态类型 derive：`AgentRunStep:170`、`PendingToolCall:194`、`ModelTurn:212` 等（:85 `use serde::{Deserialize, Serialize}`）。
- 身份字段持久化：`PendingToolCall.block_id` 要求恢复后的进程继续发消费者已见过的 id，**不重新 mint**（run/mod.rs 注释）；`preresolved_result` 也持久化。
- 不可序列化的是外层：模型值、hook、dispatcher、`ToolContext.scopes`（`#[serde(skip)]`）。因此跨进程序列化的是 **AgentRun 数据**，在新进程用新的 Agent/runner 装配后 `resume`。
- 比单进程持久化更强的记录/重放走 rig-effect-log：run 记录 effect 头（程序身份：hook 栈名、模型、策略版本）与每效果的入参/结果；replay 遇到位置不一致即 `ErrorKind::Divergence`（见 07 篇）。ECS 运行时另有场景存取（scene save/load）与身份规则（Scope/PolicyVersion/policy hash）。

## 6. 边界与坑

- Extractor 的固定 preamble 不能替换，只能 `append_preamble`。
- `Tool` 模式是尽力而为：即使 `ToolChoice::Required`，解包失败/空 submit 仍要读 `completion_calls` 与错误，预算耗尽时 best-effort 终局。
- 重试是**从头跑**不是续轮（`max_turns(1)`），别假设历史消息会保留——需要上下文显式 `.history(..)`。
- schema 来自 `schemars::JsonSchema`；手写 JSON schema 用 `output_schema_raw`，Extractor 泛型版没有这个口。
- 持久化恢复后不能指望 `ToolContext` 的 scopes（运行时句柄）还在。

## 7. 自检问题

1. Extractor 复用了哪四个既有机制，而没有新引擎？
2. 为什么默认不用 provider 原生结构化输出做提取？
3. 模型这一轮调了业务工具但没调 submit，会发生什么？usage 怎么算？
4. 哪些 run 状态可以跨进程，哪些不行？工具调用 id 如何保持连续？
5. 想用自定义手写 schema 而非 `JsonSchema`，该走 AgentBuilder 的哪个方法？
