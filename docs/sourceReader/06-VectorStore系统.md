# 06-VectorStore 系统

向量库为 Agent 提供语义检索（RAG）底座：嵌入模型把查询变向量，`VectorStoreIndex` 在后端找出最相似的 N 条。Rig 的设计是：trait 在 rig-core，实现放伴侣 crate，记忆策略在 rig-memory。

---

## 🧱 接口定义（当前签名）

```rust
// File: crates/rig-core/src/vector_store/mod.rs:133

pub trait VectorStoreIndex: WasmCompatSend + WasmCompatSync {
    /// 该后端的过滤类型
    type Filter: SearchFilter + WasmCompatSend + WasmCompatSync;

    fn top_n<T: DeserializeOwned + WasmCompatSend>(
        &self,
        req: VectorSearchRequest<Self::Filter>,
    ) -> impl Future<Output = Result<Vec<(f64, String, T)>, VectorStoreError>> + WasmCompatSend;
    // 返回 (score, id, document)

    fn top_n_ids(
        &self,
        req: VectorSearchRequest<Self::Filter>,
    ) -> impl Future<Output = Result<Vec<(f64, String)>, VectorStoreError>> + WasmCompatSend;
}
```

与旧文档的差异：**不再有关联 Error/Document**——统一 `VectorStoreError`，文档类型由调用点的 `T` 指定；查询是 `VectorSearchRequest` 结构体（`vector_store/request.rs:17`）而不是裸参数，携带 `query`（查询文本，由存储侧内部嵌入）、`samples`（top-N 数量）、`threshold`（最低相似度）、`additional_params`、`filter`（后端专属过滤式），配套 builder。

### 两个方法都要实现

仓库规则（AGENTS.md）：伴侣 crate 必须同时实现 `top_n` 与 `top_n_ids`；错误用 `VectorStoreError` 变体而不是字符串；bound 用 `WasmCompatSend/Sync`。

### 检索是 bus 上的一种效果

Agent 侧接入用 `.dynamic_context(samples, index)`：每轮折叠请求前派发 `EffectKind::Retrieve { query: RetrieveQuery::TopN { .. } }`（返回 JSON 文档，客户端侧反序列化），hook 可在 `on_dispatch` 门控（opt-in `RetrieveDispatch`）。记忆加载/落盘则是 `EffectKind::Memory`。

---

## 📦 内存实现（rig-core 内置）

`crates/rig-core/src/vector_store/in_memory_store.rs:24`：

```rust
InMemoryVectorStore::from_documents(vec![(doc, embedding)])            // id 自动 "doc{n}"
InMemoryVectorStore::from_documents_with_ids(vec![(id, doc, emb)])
InMemoryVectorStore::builder()                                          // 自选索引策略
    .index_strategy(IndexStrategy::default())   // 默认 BruteForce；可换 LSH
    .build()
```

LSH（`vector_store/lsh.rs`）：

```rust
pub struct LSH { /* dim, num_tables, num_hyperplanes */ }
LSH::new(dim, num_tables, num_hyperplanes);
lsi.hash(&vector, table_idx);           // :62  局部敏感哈希

pub struct LSHIndex { ... }
LSHIndex::new(dim, num_tables, num_hyperplanes);
index.insert(id, &embedding);           // :109
index.query(&embedding) -> Vec<String>; // :119
index.clear();                          // :141
```

---

## 🗃️ 伴侣 crate 名录（全部实现 `VectorStoreIndex`）

| crate | 后端 | crate | 后端 |
|---|---|---|---|
| `rig-qdrant` | Qdrant | `rig-milvus` | Milvus |
| `rig-sqlite` | SQLite | `rig-lancedb` | LanceDB |
| `rig-mongodb` | MongoDB | `rig-neo4j` | Neo4j |
| `rig-postgres` | Postgres | `rig-surrealdb` | SurrealDB |
| `rig-scylladb` | ScyllaDB | `rig-s3vectors` | AWS S3Vectors |
| `rig-helixdb` | HelixDB | `rig-vectorize` | Cloudflare Vectorize |
| `rig-fastembed` | 本地 Fastembed 嵌入 + 索引 | | |

经门面 feature 暴露（`rig::qdrant` ← feature `qdrant`，见 `src/lib.rs` 的 `companion_modules!` 表）。

### 伴侣 crate 模式（以 rig-qdrant / rig-sqlite 为范本）

- 独立 crate，重导出 `rig-core` 契约；
- 后端专属 builder（endpoint/key/collection…）；
- 专属 `Filter` 类型实现 `SearchFilter`；
- 服务型后端的集成测试在 `test-support/service-tests`（Docker 驱动，CI 慢车道跑）。

---

## 🧠 嵌入与检索的配合

```rust
// 1) 建索引
let embeddings: Vec<Embedding> = embedding_model.embed_texts(vec!["...".to_string()]).await?;
let mut store = InMemoryVectorStore::from_documents(
    texts.into_iter().zip(embeddings)
);

// 2) Agent 侧每轮检索
let agent = client.agent("gpt-5.2")
    .dynamic_context(5, my_vector_index)        // 每轮 top-5 进上下文
    .build();                                   // AgentBuilder::build 直接返回 Agent

// 3) 或独立检索（查询文本 + 样本数，存储内部负责嵌入）
let hits: Vec<(f64, String, MyDoc)> = store
    .top_n(VectorSearchRequest::builder().query("...").samples(5).build())
    .await?;
```

---

## 🧠 记忆与会话（memory 模块 + rig-memory）

trait 在 `crates/rig-core/src/memory.rs:98`：

```rust
pub trait ConversationMemory: WasmCompatSend + WasmCompatSync {
    fn load<'a>(&'a self, conversation_id: &'a ConversationId)
        -> WasmBoxedFuture<'a, Result<Vec<Message>, MemoryError>>;
    fn append(&self, conversation_id, messages: ...) -> ...;   // 成功轮后回写
    // 以及 owned 便捷
}
```

agent builder `.memory(backend)` + `.conversation(id)` 接入；load/append 都是 bus 效果（hook 可见、可录制）。策略族在 `crates/rig-memory/src/lib.rs`（门面 feature `memory`）：

| 策略 | 一句话 |
|---|---|
| `SlidingWindowMemory` | 按条数滑窗 |
| `TokenWindowMemory`（+ `HeuristicTokenCounter`） | 按 token 预算滑窗 |
| `PolicyMemory<M, P>` / `DemotingPolicyMemory` | 用 `NoopMemoryPolicy` 或自定义策略过滤历史 |
| `CompactingMemory<M, P, C>` + `TemplateCompactor` | 超窗时压缩成摘要 |

---

## 🔄 文档存取/检索流程图

```mermaid
graph LR
    W[写入侧] --> EMB1[EmbeddingModel.embed_texts]
    EMB1 --> INS[VectorStoreIndex.add_documents]
    Q[查询侧] --> EMB2[嵌入查询]
    EMB2 --> RET[VectorSearchRequest → top_n]
    RET --> DOCS[(score,id,doc) 列表]
    DOCS --> CTX[注入 agent 上下文 / dynamic_context]
```

---

## 🧩 总结

- 契约集中：一个 trait + 统一错误 + 请求结构体，13 个后端可插拔切换。
- 检索与记忆都走 effect bus：可被 hook 门控、可录制重放。
- 内存实现（BruteForce/LSH）零依赖可测试；服务型后端的真实验证在 service-tests。