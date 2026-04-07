---
title: "Runtime State — DeerFlow"
category: L2
parent: "[[runtime-state]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 的运行时状态以 `ThreadState` TypedDict 表达，在 LangGraph 的 `AgentState` 基础上扩展了工作区路径、产物、待办事项、上传文件等字段。状态持久化完全委托给 LangGraph checkpointer（支持内存、SQLite、Postgres 三种后端），跨 turn 的自动保存/恢复开箱即用。最具特色的是**虚拟路径抽象**：agent 看到的始终是 `/mnt/user-data/` 前缀的整洁路径，物理存储映射到 `threads/{thread_id}/` 目录，agent 逻辑与存储布局完全解耦。

## 架构分析

### ThreadState 结构

`ThreadState` 是扩展 LangGraph `AgentState` 的 TypedDict，核心字段：

| 字段 | 类型 | 用途 |
|------|------|------|
| `messages` | `list[BaseMessage]` | 对话历史（继承自 AgentState）|
| `sandbox` | `SandboxState` | 代码沙箱状态（执行环境、变量等）|
| `thread_data` | `ThreadData` | 工作区路径（workspace/uploads/outputs）|
| `title` | `str` | 当前 thread 的标题 |
| `artifacts` | `dict[str, Artifact]` | 产物注册表（文件、图表、报告）|
| `todos` | `list[Todo]` | 任务清单，跨 turn 追踪进度 |
| `uploaded_files` | `list[UploadedFile]` | 用户上传文件的元数据 |
| `viewed_images` | `set[str]` | 已查看图片路径，避免重复展示 |

### 自定义 Reducer

LangGraph 的状态合并默认是覆盖式的，DeerFlow 为两个字段定义了自定义 reducer：

- **`merge_artifacts`**：新产物与已有产物做 key 合并（同 key 更新，新 key 追加），而非整体替换，保留跨 turn 的产物积累
- **`merge_viewed_images`**：对 set 做 union 操作，确保"已查看"状态在多轮对话中单调递增

### 状态持久化（checkpointer 委托）

持久化策略完全由 LangGraph checkpointer 接管：

- **内存模式**：开发/测试用，进程重启后状态丢失
- **SQLite 模式**：单机部署，状态写入本地 SQLite 文件
- **Postgres 模式**：生产多实例部署，状态持久化到 Postgres

DeerFlow 自身无需编写序列化/反序列化逻辑，checkpointer 在每个 LangGraph 步骤后自动保存快照，恢复时按 `thread_id` 重放。

### 虚拟路径抽象

`ThreadDataMiddleware` 在 thread 首次访问时创建物理目录结构 `threads/{thread_id}/`，并维护虚拟路径到物理路径的映射：

```
虚拟路径（agent 侧）          物理路径（文件系统）
/mnt/user-data/workspace/  →  threads/abc123/workspace/
/mnt/user-data/uploads/    →  threads/abc123/uploads/
/mnt/user-data/outputs/    →  threads/abc123/outputs/
```

Agent 所有文件操作使用虚拟路径，`sandbox` 层在执行前做路径替换。这使 agent 的文件操作逻辑与具体 thread 的物理存储位置彻底解耦。

### 关键代码路径

- `deerflow/agents/thread_state.py` — `ThreadState` TypedDict 定义、`merge_artifacts`、`merge_viewed_images` reducer
- `deerflow/agents/middlewares/thread_data_middleware.py` — thread 目录创建、虚拟路径到物理路径的映射维护
- `deerflow/sandbox/` — 沙箱层的虚拟路径替换逻辑

## 设计亮点

- **虚拟路径抽象**：agent 代码中无任何 thread-specific 路径硬编码，同一 agent 逻辑可在任意 thread 中运行，测试时可轻松替换物理路径
- **checkpointer 委托的持久化**：无需自实现序列化层，LangGraph checkpointer 提供原子性保存 + 多后端支持，开发者只需切换配置即可升级存储策略
- **自定义 reducer 的精细合并**：产物和已查看图片用语义化合并而非覆盖，避免多 turn 对话中后续状态覆盖前序积累
- **`todos` 字段的跨 turn 进度追踪**：将任务清单作为一等状态字段，agent 可显式更新 todo 状态，任务进度在多轮对话中持续可见

## 局限性

- **State schema 固定**：`ThreadState` 是 TypedDict，自定义 agent 无法在不修改核心代码的情况下追加自定义字段，扩展性受限
- **无响应式状态传播**：状态变化不会触发订阅回调（如 Claude Code 的 MobX observable），UI 层需主动轮询或依赖 LangGraph streaming 机制获取更新
- **checkpointer 一致性边界模糊**：中间件内的副作用（如创建目录）在 checkpointer 保存之外执行，若 checkpointer 保存失败，文件系统侧变更不会自动回滚
- **viewed_images 内存开销**：`viewed_images` 是 set，随对话轮次无上界增长，长对话场景下可能累积大量路径字符串

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
