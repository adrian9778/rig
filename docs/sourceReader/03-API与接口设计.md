# 03-API 与接口设计

Rig 项目提供了一套清晰且一致的 API，使得开发者可以轻松地调用各种 AI 模型、嵌入向量计算以及管理工具和向量库。该文档深入解释了所有对外暴露的函数、结构体和 trait 的用途及使用方法。

---

## 🧠 接口总览

Rig 提供了几类核心接口，涵盖模型交互、向量搜索、任务调度等，分别对应不同的抽象层：

| 抽象层        | 主要接口                        | 文件路径 |
|---------------|----------------------------------|----------|
| 模型访问      | `CompletionModel`, `EmbeddingModel` | crates/rig-core/src/model/ |
| 向量检索      | `VectorStoreIndex`               | crates/rig-core/src/vector_store/ |
| 工具管理      | `Tool`, `ToolExecutor`           | crates/rig-core/src/tool/ |
| 任务执行      | `Agent`、生命周期钩子（Hooks）   | crates/rig-agent/src/ |

---

## 💬 API 核心结构体与函数

### 1. Client (`rig-core`)

#### 构造器

```rust
// File: crates/rig-core/src/client.rs

impl<Ext, H> Client<Ext, H>
where
    Ext: CompletionModel + EmbeddingModel + VectorStoreIndex,
{
    pub fn new(model: Ext) -> Self { 
        Self {
            model,
            http_client: reqwest::Client::new(),
        }
    }

    // 从环境变量创建
    pub fn from_env() -> Result<Self, RigError> {
        // 这些功能通常在具体 Provider 实现中定义
        todo!("Implementation in specific provider")
    }

    // 从配置值构造
    pub fn from_val(val: impl Into<ProviderConfig>) -> Result<Self, RigError> {
        todo!("Implementation in specific provider")
    }
}
```

#### 完整使用示例

```rust
// File: examples/client_usage.rs

use rig_openai::OpenAI;
use rig::agent::{Client, Agent};

let client = Client::new(OpenAI::default());
// 或者构建自定义配置
let client = Client::new(
    OpenAI::builder()
        .api_key("sk-...") // 从环境变量加载
        .base_url("https://api.openai.com/v1")
        .build()
);
```

---

### 2. Agent Builder (`rig-agent`)

Agent 的创建过程使用 Builder 模式：

```rust
// File: crates/rig-agent/src/agent.rs

pub struct AgentBuilder<Ext, H = reqwest::Client> {
    client: Client<Ext, H>,
    preamble: Option<String>,
    tools: Vec<Box<dyn Tool>>,
    temperature: f64,
    max_tokens: Option<u32>,
    stop_sequences: Vec<String>,
    hooks: Vec<Box<dyn AgentHook>>,
    // 其他可用参数...
}

impl<Ext, H> AgentBuilder<Ext, H> {
    pub fn preamble(self, preamble: impl Into<String>) -> Self {
        Self {
            preamble: Some(preamble.into()),
            ..self
        }
    }

    pub fn tool<T: Tool + 'static>(self, tool: T) -> Self {
        let mut tools = self.tools;
        tools.push(Box::new(tool));
        Self {
            tools,
            ..self
        }
    }

    pub fn temperature(self, temp: f64) -> Self {
        Self {
            temperature: temp,
            ..self
        }
    }

    pub fn max_tokens(self, max_tokens: Option<u32>) -> Self {
        Self {
            max_tokens,
            ..self
        }
    }

    pub fn stop_sequences(self, sequences: Vec<String>) -> Self {
        Self {
            stop_sequences: sequences,
            ..self
        }
    }

    pub fn hooks(mut self, hooks: Vec<Box<dyn AgentHook>>) -> Self {
        self.hooks = hooks;
        self
    }

    pub fn build(self) -> Agent<Ext, H> {
        Agent {
            client: self.client,
            preamble: self.preamble,
            tools: self.tools,
            temperature: self.temperature,
            max_tokens: self.max_tokens,
            stop_sequences: self.stop_sequences,
            hooks: self.hooks,
        }
    }
}
```

#### 示例：

```rust
// File: examples/agent_builder.rs

let agent = client
    .agent("gpt-4")
    .preamble("You are a helpful assistant.")
    .tool(weather_tool)
    .temperature(0.8)
    .max_tokens(Some(1024))
    .stop_sequences(vec!["\n\n".to_string()])
    .build();
```

---

### 3. 完整运行 API（同步/异步）

#### run()

```rust
// File: crates/rig-agent/src/agent.rs

impl<Ext, H> Agent<Ext, H> {
    pub async fn run(&self, prompt: impl Into<String>) -> Result<String, RigError> {
        let mut history = Vec::new();
        
        // 构造请求
        let mut request = CompletionRequest::default();
        if let Some(preamble) = &self.preamble {
            request.extra_context.push(preamble.clone());
        }
        request.messages.push(Message::user(prompt.into()));

        // 执行Hook处理（前置）
        let patched_request = self.apply_hooks_before_completion(&request).await?;
        
        // 调用模型
        let response = self.client.model.complete(patched_request).await.map_err(RigError::from)?;
        
        // 解析结果
        let text = self.extract_text_from_response(&response).await?;
        
        // 执行Hook处理（后置）
        self.apply_hooks_after_response(&response).await?;
        
        Ok(text)
    }
}
```

用于执行单轮对话或任务，返回 AI 回答。

```rust
let response = agent.run("What is 2+2?").await?;
println!("Answer is: {}", response);
```

#### stream()

```rust
// File: crates/rig-agent/src/agent.rs

impl<Ext, H> Agent<Ext, H> {
    pub async fn stream(&self, prompt: impl Into<String>) -> Result<impl Stream<Item = String>, RigError> {
        let mut request = CompletionRequest::default();
        if let Some(preamble) = &self.preamble {
            request.extra_context.push(preamble.clone());
        }
        request.messages.push(Message::user(prompt.into()));

        // 应用前置Hook
        let patched_request = self.apply_hooks_before_completion(&request).await?;
        
        // 调用模型流式输出
        let stream = self.client.model.complete_stream(patched_request).await.map_err(RigError::from)?;
        
        Ok(stream.map(|chunk| chunk.text))
    }
}
```

流式输出生成结果（适用于长文本场景）：

```rust
let mut stream = agent.stream("Explain quantum computing").await?;
while let Some(chunk) = stream.next().await {
    print!("{}", chunk);
}
```

---

## 🔄 Trait 接口详解

### 1. `CompletionModel` 特性

```rust
// File: crates/rig-core/src/model/completion.rs

pub trait CompletionModel: Send + Sync {
    type Error: std::error::Error + Send + Sync + 'static;
    type Response: ProviderResponse<Completion = Self::Completion>;
    type Completion: CompletionResponse;

    async fn complete(&self, request: CompletionRequest) -> Result<Self::Response, Self::Error>;
    
    // 可选：流式方法
    async fn complete_stream(
        &self,
        request: CompletionRequest
    ) -> Result<impl Stream<Item = Self::Completion>, Self::Error> {
        // 默认实现：抛出不支持错误
        Err(UnsupportedStreamError::new().into())
    }
}
```

#### 关键组件

- `request`: `CompletionRequest` 是输入请求结构体，定义在 `crates/rig-core/src/model/completion.rs`
- `response`: 包含了模型返回的文本、工具调用信息等
- `completion`: 提供完成响应具体字段访问接口，定义在 `crates/rig-core/src/model/mod.rs`

> 实现者示例：OpenAI、Gemini 等不同 Provider 均需实现此接口

### 2. `EmbeddingModel` 特性

```rust
// File: crates/rig-core/src/model/embedding.rs

pub trait EmbeddingModel: Send + Sync {
    type Error: std::error::Error + Send + Sync + 'static;
    type Response: ProviderResponse<Embedding = Self::Embedding>;
    type Embedding: EmbeddingResponse;

    async fn embed(&self, request: EmbeddingRequest) -> Result<Self::Response, Self::Error>;
}
```

#### 用途

- 将自然语言转换为向量
- 支持语义相似性判断、检索等下游任务

### 3. `VectorStoreIndex` 特性

```rust
// File: crates/rig-core/src/vector_store/mod.rs

pub trait VectorStoreIndex {
    type Error: std::error::Error + Send + Sync + 'static;
    type Document: VectorDocument;

    async fn top_n(&self, query: &Embedding, n: usize) -> Result<Vec<Self::Document>, Self::Error>;
    async fn top_n_ids(&self, query: &Embedding, n: usize) -> Result<Vec<DocId>, Self::Error>;
}
```

#### 用例详情

```rust
// File: examples/vector_store_usage.rs

let embedding_request = EmbeddingRequest {
    input: vec!["What is the weather today?".to_string()],
};

let embedding_response = client.embed(embedding_request).await?;
let embeddings = embedding_response.embeddings;

// 将向量传入 VectorStore
let result_documents = vector_store.top_n(&embeddings[0], 5).await?;
for doc in result_documents {
    println!("Found document: {:?}", doc);
}
```

支持向量检索，并返回最匹配的几条结果。

### 4. `Tool` 特性

```rust
// File: crates/rig-core/src/tool/mod.rs

pub trait Tool {
    type Error: std::error::Error + Send + Sync + 'static;
    type Output: serde::Serialize;

    async fn call(&self, input: &str) -> Result<Self::Output, Self::Error>;
}
```

#### 使用说明

开发者只需实现 `Tool` 接口即可注册进 Agent，用于执行真实任务。

示例：

```rust
// File: examples/weather_tool.rs

#[derive(Tool)]
pub struct WeatherTool {}

impl Tool for WeatherTool {
    type Error = Box<dyn std::error::Error + Send + Sync>;
    type Output = String;

    async fn call(&self, input: &str) -> Result<Self::Output, Self::Error> {
        // 实现实际的工具逻辑
        Ok("Sunny".to_string())
    }
}
```

---

## 🛠️ 工具宏实现（`rig-derive`）

为了简化工具定义，Rig 提供了一个 derive macro：

```rust
// File: crates/rig-derive/src/tool.rs

#[proc_macro_derive(Tool)]
pub fn tool_derive(input: TokenStream) -> TokenStream {
    let ast: DeriveInput = syn::parse(input).unwrap();
    
    // 自动生成 Tool 实现代码...
    // 包括错误处理、调用逻辑等
    todo!("Auto-generated implementation")
}
```

使用 `#[derive(Tool)]` 可以自动为结构体添加必要的 Trait 方法，无需人工实现。

---

## 🧠 接口调用示例与详细流程

### 1. 完整的 Agent 构建流程

```rust
// File: examples/complete_example.rs

use rig_openai::OpenAI;
use rig_agent::{Agent, Client};
use rig_agent::hooks::LoggingHook;

let client = Client::new(OpenAI::default());

// 定义工具
let weather_tool = WeatherTool {};
let calendar_tool = CalendarTool {};

// 构建 Agent
let hooks = vec![Box::new(LoggingHook {}) as Box<dyn AgentHook>];

let agent = client
    .agent("gpt-4")
    .preamble("You are a helpful assistant.")
    .tool(weather_tool)
    .tool(calendar_tool)
    .temperature(0.7)
    .max_tokens(Some(1024))
    .stop_sequences(vec!["\n\n".to_string()])
    .hooks(hooks)
    .build();
```

### 2. 请求处理流程追踪

```rust
// File: crates/rig-agent/src/agent.rs

impl<Ext, H> Agent<Ext, H> {
    async fn handle_request(&self, prompt: String) -> Result<String, RigError> {
        // 步骤1: 构建标准 request 结构
        let mut request = CompletionRequest::default();
        request.messages.push(Message::user(prompt));
        
        // 步骤2: 应用前置Hook处理
        let patched_request = self.apply_hooks_before_completion(&request).await?;
        
        // 步骤3: 实际调用模型方法
        let response = self.client.model.complete(patched_request).await?;
        // 步骤4: 解析响应
        let text = self.extract_text_from_response(&response).await?;
        // 步骤5: 应用后置Hook处理
        self.apply_hooks_after_response(&response).await?;
        
        Ok(text)
    }
}
```

---

## 🧩 总结

Rig 的接口设计非常重视开发者体验，采用：

- **Builder 模式** 灵活配置
- **抽象 Trait 接口** 实现多Provider适配
- **结构化数据** 明确输入输出定义
- **标准异步支持** 便于构建高性能应用

通过这些接口，开发者可以快速搭建功能强大且易于维护的 AI 应用系统。

---

## 📁 文件路径映射表

| 功能 | 文件路径 |
|------|----------|
| Core traits | crates/rig-core/src/model/completion.rs, crates/rig-core/src/model/embedding.rs, etc. |
| Agent Runner | crates/rig-agent/src/agent.rs |
| Hook system | crates/rig-agent/src/hooks.rs |
| Tool macro | crates/rig-derive/src/tool.rs |
| Client setup | crates/rig-core/src/client.rs, provider-specific crate implementations |