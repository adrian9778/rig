# 07-Agent 与钩子系统

Agent 是 Rig 的工作单元：围绕模型与工具集执行多轮任务。钩子（`AgentHook`）是 per-run 的生命周期观察者与转向器。本篇给出**全部**事件、动作、组合语义，以及 rig-ecs 这一替代运行时。

---

## 🧠 架构分层回顾

- `Agent`（构造面）→ `AgentRunner`（hook 感知驱动器，`agent/runner.rs:56`）→ `AgentRun`（sans-IO 状态机，`run/mod.rs:445`）→ bus（`bus/mod.rs`）。
- `run` 与 `stream` 共享同一驱动循环，**事件在两个面上行为完全一致**；`run_channel` 把 run 拆成 future + `RunEvents`。

---

## 🔗 AgentHook：每个事件一个方法（`agent/hook.rs:1261`）

```rust
pub trait AgentHook: WasmCompatSend + WasmCompatSync {
    fn name(&self) -> Option<String> { None }   // 依赖携带状态决策的 hook 必须命名（重放身份）
    // —— 运行边界 ——
    fn on_run_start(&self, ctx, event: RunStart<'_>) -> Future<RunStartAction>;
    fn on_run_settled(&self, ctx, event: RunSettled<'_>) -> Future<()>;   // 观察 only，恰一次
    // —— 模型边界 ——
    fn on_model_select(&self, ctx, event: ModelSelection<'_>) -> ModelSelectionAction;  // 同步
    fn on_completion_call(&self, ctx, event: CompletionCall<'_>) -> Future<CompletionCallAction>;
    fn on_model_turn_finished(&self, ctx, event: ModelTurnFinished<'_>) -> Future<ModelTurnAction>;
    // —— 无效调用 ——
    fn on_invalid_tool_call(&self, ctx, event: &InvalidToolCallContext) -> Future<Option<InvalidToolCallAction>>;
    // —— 增量（观察 only）——
    fn on_text_delta / on_reasoning_delta / on_tool_call_delta -> Future<ObservationAction>;
    // —— 总线边界 ——
    fn on_dispatch(&self, ctx, event: DispatchEvent<'_>) -> Future<DispatchAction>;
    fn on_outcome(&self, ctx, event: OutcomeEvent<'_>) -> Future<OutcomeAction>;
    // —— 观察开关 ——
    fn observes(&self, kind: StepEventKind) -> bool;
}
```

### `HookContext`（hook.rs:318）

run 作用域：`run_id`、`turn`（AtomicUsize）、`is_streaming`、`agent_name`、共享 `Scratchpad`（hook 间读写）、`entries`（本 run 可见的全部 `RunEntry`）与 `pending_entries`、总线 `Dispatcher`、被 Deny 拦下时抢救的补丁（`salvaged_patches`，嵌套栈也保得住）。

### 事件 × 动作对照表

| 事件 | 动作枚举 | 变体 | 组合语义 |
|---|---|---|---|
| `on_run_start` | `RunStartAction` | `Continue` / `Rewrite(Message)` / `Stop(String)` | 改写按注册序链式传递；第一个 Stop 终止，**在任何 provider 调用前** |
| `on_model_select` | `ModelSelectionAction` | `Continue` / `Select(ModelRef)` / `Stop(String)` | 最后的选择生效；同步、非阻塞；in-flight 尝试不换绑 |
| `on_completion_call` | `CompletionCallAction` | `Continue` / `Patch(RequestPatch)` / `Stop(String)` | **补丁按注册序合并**（见下）；停止短路 |
| `on_model_turn_finished` | `ModelTurnAction` | `Continue` / `Retry(Repeat\|Feedback)` / `Stop(String)` | Retry 仅对无工具轮合法、消耗总预算；Retry/Stop 短路剩余 hook |
| `on_invalid_tool_call` | `Option<InvalidToolCallAction>` | `Fail` / `Retry{feedback}` / `Repair{tool_name}` / `Skip{reason}` / `Stop{reason}` | 返回 None 交给后面的 hook；全 None = 保持 fail-fast |
| `on_text_delta` / `on_reasoning_delta` / `on_tool_call_delta` | `ObservationAction` | `Continue` / `Stop(String)` | 观察；流式增量在轮被接受前是暂定的 |
| `on_dispatch` | `DispatchAction` | `Proceed` / `Patch(EffectKind)` / `Deny(ErrorReport)` | 每个效果都过这里；Patch 必须保家族（工具调用只可改参数）；Deny=效果以失败报告解决：工具 `Cancelled` 取消 run、其余成 skipped 结果；补丁保存在 `salvaged_patches` |
| `on_outcome` | `OutcomeAction` | `Proceed` / `Replace(Result<Outcome, ErrorReport>)` | 后一个 hook 看到的是前一个 Replace 后的值；`stop()`=Cancelled 替换，短路嵌套栈 |
| `on_run_settled` | `()` | — | 观察 only：结局已定；run 的错误**先**过这里再到达消费者 |

### `RequestPatch` 合并规则（`run/patch.rs`）

| 字段 | 合并 |
|---|---|
| `extra_context: Vec<Document>` | **追加** |
| `additional_params: Value` | **浅合并**（对象键覆盖） |
| `active_tools: Option<Vec<String>>` | **交集**（收窄） |
| `preamble` / `temperature` / `max_tokens` / `tool_choice` / `history` | **后写者赢** + `tracing::warn!` |

补丁**只对当轮生效，不粘滞**。

### `observes` 门（hook.rs:1433）

内部家族 `Memory`/`Retrieve`/`Embed`/`Rerank`/`Custom` 的派发默认**不可见**（没有 hook 在总线之前见过它们）；hook 想门控就在 `observes` 里对相应 `StepEventKind` 返回 true。对 `CompletionDispatch`/`ToolDispatch` 返回 false 是**硬门**——该 hook 对这些派发不再被调用（也不能再 Deny）。只想去掉增量噪音的覆写应只对 delta 类返回 false。

### 注册顺序

**观察型 hook 注册在转向型之前**（Stop 动作会短路）。`.add_hook(h)`（builder.rs:411）或 runner 上 `.add_hook(h)`（runner.rs:131）；`HookStack`（hook.rs:1594）记录每个 hook 的名字（`name()` 或类型名）——这是 effect 日志 header 里"程序"的一部分，嵌套栈展平。

---

## 🧩 现实示例（示例目录 `examples/`）

- `agent_with_retry_hook`：`on_model_turn_finished` 返回 `retry_with_feedback`。
- `request_hook`：`on_completion_call` 注入 RequestPatch。
- `agent_with_approval_policy` / `agent_with_durable_approval`：`on_dispatch` Deny + 外部审批。
- `agent_run_stepping`：直接逐步驱动 `AgentRun`。

---

## 🧪 测试

`test_utils`（门面 feature `test-utils`）提供构造 agent 的假设施；hook 语义本身由 `agent/hook/tests.rs`、`agent/engine/tests.rs` 与 loom 模型检查（`--cfg rig_loom` 选 `sync.rs` 的模型 shim）钉住。**每个 hook 语义必须在 stream 与 run 两个面上一致**——这是 CI 断言，不是约定俗成。

---

## 🧬 rig-ecs：运行即图（替代运行时）

`crates/rig-ecs/`（未发布，`publish=false`，依赖 `bevy_ecs` + `bevy_tasks`，feature：`replay`(默认)/`reflect`/`assets`）。**不依赖 rig-agent**，直接消费 rig-core 的 effect/serve 契约。

### bus 即插件（`src/bus/mod.rs`）

- **效果是实体**：`commands.spawn(PendingEffect{key,kind})` → `InFlight`（handler 接走）→ `EffectOutcome`（落定）；流式在 `Streamed` 组件累积。就绪即组件落地（`Added<EffectOutcome>`、`On<Add,..>` observer）。
- **handler 是实体**：`Handlers::register("model", handler)` spawn 一个带 `Bound`（key+descriptor，serde）的实体，擦除 handler 进 `HandlerTable` 资源。
- **driver 是两个系统**：`BusSet::{Gate, Dispatch, Collect, Judge}` 四个集；Dispatch 把 handler future 派到 bevy 任务池（存成效果实体自己的 `Task`），Collect 落定结果。没有 `block_on`（守卫 grep）。
- **因果是 `ChildOf`**：handler 即系统可 spawn 子效果；despawn 即取消（任务 drop、记录 `Cancelled`，Bevy 级联清子孙）。
- **截获是两个槽**：`Gate` 里用户系统 Patch/Deny/`Held`（按住等下个 tick）；`Judge` 在记录关闭后改写 `EffectOutcome`。
- **host 每 tick 调 `run_to_quiescence`**（plugin.rs:185，上限 `QUIESCENCE_CAP=64` pass）。

### 运行即图（`src/agent/mod.rs`）

| 概念 | 载体 |
|---|---|
| Agent | 实体 + `Owner`/`Preamble`/`Temperature`/`MaxTokens`/`UsesModel`/`Grant`（工具链接实体）/`Context`/`Route` |
| Run | `Run` + `RunOf` + 相位标记（`Assembling`/`AwaitingModel`/`Settled`/`Failed`…）+ `RunResult`/`Usage` |
| Turn | `Turn`，`ChildOf` run；`Advert`/`Attachment` 链接实体 |
| 话语 | `Utterance` + `Role` + 有序内容部件子实体（`Order`） |
| 转向 | 用户系统写组件：run 上的 `Cancelled`、turn 上的 `Retry`/`RequestPatch`、无效调用上的 `Resolution`、run 上的 `UsesModel` |

**一次 fold**：`policy::fold_request`（policy/mod.rs:217）是全 crate 唯一从图构造 `CompletionRequest` 的函数（根守卫拒绝第二个）。系统集顺序（systems/mod.rs）：`Advance → Select → Assemble → Patch(转向槽②) → [bus Gate/Dispatch/Collect/Judge] → Fold → Judge → Materialise → Checkpoint → Settle`。

### 场景与重放

- `agent::scene::{save_world, load_world}`（scene.rs:287/:483）：保存库支持的图/效果状态（不是任意 world）；`SceneExtensions` 在两个 world 注册应用组件（版本化名字，未注册名字拒绝加载）；资源、自定义效果组件、任务归 host。**先绑 handler 再 load，后装插入 observer**。
- `replay::stamp_run`（replay/mod.rs:354）：写入 run 的确切 `Scope` 的 `ProgramIdentity{policy,required_row}`；应用必须为自定义系统声明非空 `agent::PolicyVersion`，缺失即"未验证"。
- `replay::check_replayable`（:389）：在绑好 replayer 的新 world 里检查——精确 Scope 匹配、policy hash 相等、required row 无差异；不搜索其他匹配 hash。中间件要原样重装：replayer 只供给内层 handler 的录制交换。

### 最小用法（examples/hello_model.rs，30 行用户代码）

```rust
let mut app = App::new();
rig_ecs::bus::Bus::with_policy(ServingPolicy::default()).install(app.world_mut());
app.add_plugins(ScheduleRunnerPlugin::default())
    .add_systems(Update, run_to_quiescence)          // host 拥有循环
    .add_systems(Startup, (register_the_model, ask).chain())
    .add_observer(print_the_answer)                  // On<Add, EffectOutcome>
    .run();
```

### 跨运行时对齐（tests/ecs_parity/ + tests/providers/*/cassette/ecs_*.rs）

逐 provider wire 比较经典与 ECS 两套运行时：OpenAI Chat / OpenAI Responses / Anthropic / Gemini / DeepSeek / Doubleword / Venice / OpenRouter / xAI 等。矩阵覆盖长工具环、故障注入、图像、推理块、流投递（`StreamItemsDelivered`、双消费者、五线矩阵）、检查点边界；native 身份期望钉在 `test-support/rig-test-support/src/ecs_goldens/identities.json`。

---

## 🧩 总结

- 钩子是**每事件一方法**的编译期完备面：不支持的组合直接编不过。
- 效果派发是唯一干预点：完成调用可打补丁，工具调用可拒，结果可替换；记忆/检索等内部家族要 opt-in。
- 流式与非流式共享语义；`RunSettled` 保证 run 错误先被观察再交给消费者。
- rig-ecs 把同一套 effect 协议搬进 Bevy World：干预=写组件，持久化=场景，重放=身份校验后的日志回放。