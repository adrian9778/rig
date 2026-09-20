# Rig 项目文档库索引

本目录为 **Rig** 项目的源码级技术文档库，旨在帮助开发者深入理解项目架构与核心组件，并可基于此文档进行代码复现或重构。

> 同步基线：工作区版本 **0.42.0** 之后、下一次发布之前的 main 分支（2026-09 核对）。
> 该区间包含大量破坏性重构（Provider 泛型客户端、传输层拆分、effect bus、rig-ecs 等），对应 `MIGRATING.md` 的 "0.41 → next" 章节。所有类型签名与文件路径均与当前代码逐条核对。

---

## 📁 路径结构概览

```bash
docs/sourceReader/
├── 00-TOC.md                    # 本文件
├── 01-架构总览.md               # 工作区拓扑、核心抽象、一次请求的完整旅程
├── 02-核心模块与类关系.md         # rig-core / rig-agent / rig-ecs 的模块地图与类型关系
├── 03-API与接口设计.md           # 客户端、模型、Agent、工具、向量库的对外 API
├── 04-配置与数据流.md             # 配置体系（AgentConfig、feature 矩阵）与运行时数据流
├── 05-Provider 体系说明.md        # Provider trait 家族、26 个内置 Provider、新增 Provider 指南
├── 06-VectorStore 系统.md         # 向量库 trait、内存实现、12+ 伴侣 crate、记忆策略
├── 07-Agent 与钩子系统.md         # AgentHook 全事件、动作语义、HookStack 规则、rig-ecs 运行时
├── 08-错误与异常处理.md           # ErrorReport 线错误模型、各域错误枚举、可重试语义
└── 09-构建与运行说明.md           # 工具链、测试布局、cassette、xtask 验证、发布说明
```

---

## 📘 各篇章说明

### 🧠 01-架构总览.md
> **目的**：介绍工作区的 crate 拓扑（facade `rig`、无传输依赖的 `rig-core`、经典运行时 `rig-agent`、Bevy World 运行时 `rig-ecs`），以及贯穿两者的 effect 协议。
>
> **内容包括**：
> - 工作区 crate 依赖图与职责表
> - 核心抽象（CompletionModel、EmbeddingModel、VectorStoreIndex、Tool/PortableTool、effect bus）
> - 一次带工具的多轮请求的完整旅程

### 🔗 02-核心模块与类关系.md
> **目的**：从模块角度梳理 `Client<P, H>` 泛型客户端、Provider/Has* 能力 trait、Agent/AgentRunner/AgentRun 状态机、bus 三角色与 rig-ecs 的系统集。
>
> **内容包括**：
> - rig-core 与 rig-agent 的模块地图（真实路径）
> - 泛型客户端架构与 builder 状态机
> - sans-IO 运行状态机与 hook 栈
> - Mermaid 类图 / 依赖图

### ⚙️ 03-API与接口设计.md
> **目的**：展示对外 API 的真实签名与用法：客户端构造、completion/embedding 调用、Agent 的 run/stream/run_channel/chat/resume、提取器、工具定义。
>
> **内容包括**：
> - 各 API 的真实代码签名（文件:行号可定位）
> - `#[rig_tool]` 宏与上下文工具
> - 多轮流式事件（MultiTurnStreamItem / RunEvents）

### 🧭 04-配置与数据流.md
> **目的**：解释配置项来源（builder → 共享 AgentConfig → runner 副本）、RequestPatch 合并语义、feature 门控矩阵。
>
> **内容包括**：
> - 配置流转图
> - AgentRunner 一次 run 的逐步数据流
> - facade feature 与环境变量

### 🌐 05-Provider 体系说明.md
> **目的**：详解 Provider 的编写模型（一个 `Provider` 值类型 + 若干 `Has*` 能力实现）、内置 Provider 名录、共享适配层与 cassette 回归测试。
>
> **内容包括**：
> - `providers/internal/` 适配层（36 个文件）
> - 请求/响应转换与错误保留
> - 新增 Provider 的步骤清单

### 🧠 06-VectorStore 系统.md
> **目的**：描述向量库 trait（Filter 关联类型 + VectorSearchRequest）、内存实现与 LSH、各伴侣 crate 与记忆策略 crate。
>
> **内容包括**：
> - trait 签名与调用流程
> - InMemoryVectorStore / LSHIndex
> - rig-memory 策略族

### 🧩 07-Agent 与钩子系统.md
> **目的**：完整解析 AgentHook 的每个生命周期事件、动作枚举与 HookStack 组合语义；并介绍 rig-ecs 这一"运行即图"的替代运行时。
>
> **内容包括**：
> - 事件 × 动作对照表（on_run_start/on_model_select/on_dispatch/…）
> - 补丁合并与短路规则
> - rig-ecs：效果即实体、场景存取、重放身份

### 🛑 08-错误与异常处理.md
> **目的**：梳理线错误模型 `ErrorReport`/`ErrorKind`、各域错误枚举、可重试判定与 provider 响应保留。
>
> **内容包括**：
> - 唯一的状态码→可重试表
> - PromptError / CompletionError / VectorStoreError 等
> - 错误跨边界的转换链

### 🏗️ 09-构建与运行说明.md
> **目的**：帮助开发者正确编译、测试和验证项目（Rust 1.95.0 工具链、nextest、cassette、xtask）。
>
> **内容包括**：
> - 工作区成员与测试布局规则
> - cassette 录制/回放命令
> - `cargo xtask verify` 与发布文档的生成规则