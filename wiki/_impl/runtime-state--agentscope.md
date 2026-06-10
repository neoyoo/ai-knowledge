---
title: "runtime-state——agentscope"
category: L2
parent: "[[runtime-state]]"
source: "agentscope"
source_version: "v2.0.1-11-g0e5418e8"
concept: "runtime-state"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的 runtime state 以 `AgentState` 为中心，并由 App runtime 的 `SessionRecord`、`StorageBase`、`MessageBus`、`WorkspaceManager` 共同定义生命周期。它不再使用旧版 `_ConfigCls + StateModule + SessionBase` 三层状态树；当前源码中未找到 `src/agentscope/module/_state_module.py` 或 `src/agentscope/session/`。

更准确地说，AgentScope 2.x 把 agent 状态分成两类：

- **可持久化 session state**：`AgentState`，随 `SessionRecord` 存储和恢复
- **可重建 runtime dependencies**：model、toolkit、middlewares、workspace、message bus，每轮由 `ChatService` 从配置和 managers 重建

---

## 架构分析

### AgentState

`AgentState` 是一棵 Pydantic state：

```
AgentState
├── session_id
├── summary
├── context: list[Msg]
├── reply_id
├── cur_iter
├── permission_context: PermissionContext
├── tool_context: ToolContext
│   ├── read_file_cache
│   └── activated_groups
└── tasks_context: TaskContext
```

它覆盖了主循环、context、工具、权限、task 这几个需要跨 run 保留的状态。

### SessionRecord

`SessionRecord` 把 state 和不可变/低频变更配置放在一起：

- `config.workspace_id`
- `config.chat_model_config`
- `config.fallback_chat_model_config`
- `source` / `source_schedule_id`
- `team_id`
- `state`

这让 session 不只是聊天历史 id，而是“这个 agent 在哪个 workspace、用哪个模型、属于哪个 team、当前状态是什么”的运行单元。

### Runtime dependencies 每轮重建

`ChatService._run_impl()` 会每轮重建：

- workspace：`workspace_manager.get_workspace(...)`
- toolkit：workspace tools + planning + background + schedule + team + extras + skills + MCPs
- middlewares：inbox、state change、tool offload、extra middlewares
- model / fallback model
- agent：`Agent(..., state=session_record.state, ...)`

这意味着 runtime state 不试图序列化工具对象、模型 client、连接池、middleware 实例，而是保存足够的 ID/config 后重建。

### ToolContext

`ToolContext` 是 `AgentState` 中最容易被忽略的部分：

- `activated_groups`：当前模型可见的非 basic 工具组
- `read_file_cache`：Read/Edit 等文件工具的缓存，带 `updated_at` 校验和 LRU/size 控制

context 压缩后，Agent 会清理不再被保留 context 引用的 read cache，避免状态长期膨胀。

---

## 关键代码路径

- `src/agentscope/state/_state.py` — `AgentState` / `ToolContext` / `TaskContext`
- `src/agentscope/app/storage/_model/_session.py` — `SessionRecord`
- `src/agentscope/app/storage/_model/_agent.py` — `AgentData`
- `src/agentscope/app/_service/_chat.py` — runtime dependency assembly
- `src/agentscope/app/_service/_toolkit.py` — toolkit assembly
- `src/agentscope/app/message_bus/_base.py` — run lock、events、inbox、wakeup
- `src/agentscope/agent/_agent.py` — state mutation and loop control

---

## 设计亮点

### 1. State 只保存可恢复事实，不保存活对象

模型 client、MCP session、workspace 实例、middleware 都不进入 state。恢复时通过 `SessionConfig` 和 managers 重建，避免序列化活连接和文件句柄。

### 2. `AgentState` 同时覆盖 local 和 Web runtime

本地代理可以直接持有 `AgentState`；Web runtime 可以把它放进 `SessionRecord` 后端。这个边界比“只保存 messages”更完整，又比“序列化整个 agent 对象”更稳。

### 3. Tool active groups 是状态，不是 toolkit 属性

工具组激活状态跟 session 走，而不是跟 `Toolkit` 走。每轮重建 toolkit 后仍可根据 `state.tool_context.activated_groups` 计算模型可见工具。

### 4. PermissionContext 与 workspace 联动

`ChatService` 每轮把 workspace workdir 注入 `state.permission_context.working_directories`。权限状态不是工具局部变量，而是 session runtime state 的一部分。

### 5. Inbox/wakeup 将 idle session 纳入 runtime

`MessageBus` 把 inbox、wakeup 和 session lock 统一起来，使“当前没有进程持有 agent 对象”的 session 也可以接收 team/background/schedule 事件。

---

## 局限性

### 1. AgentState 仍是较粗粒度整体写入

每轮结束后 `update_session_state()` 写整个 `AgentState`。长 context 或复杂 task state 会带来线性序列化/网络开销。

### 2. Runtime dependency 重建依赖外部配置一致

恢复能否成功不仅看 `AgentState`，还取决于 agent record、session config、workspace manager、credentials、MCP server 是否仍可用。

### 3. Local workspace 的 tenant 隔离不足

Local workspace manager 当前丢弃 `user_id/session_id`，workdir 为 `basedir/agent_id`。同一 agent 的不同 session/workspace 共享目录，对 Web 多租户形态要额外防护。

### 4. 没有独立 state migration 层

`AgentState` 字段变化依赖 Pydantic 默认值和 storage 后端兼容，没有专门 schema version/migration pipeline。

---

## 对 agent-os 的借鉴

agent-os 可以采用同样的“state vs dependencies”边界：

- `AgentState`：只放 context summary、active context、tool context、permission context、tasks
- `SessionConfig`：workspace id、model config、policy profile、agent template id
- `AgentAssembler`：从 config + state + managers 重建 agent
- `RuntimeManagers`：workspace、tool providers、message bus、credential resolver

本地代理实现可以把 managers 简化为本地对象；Web 分布式实现应保证 managers 可跨进程重建同一 session。

---

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
- 核心文件：
  - `src/agentscope/state/_state.py`
  - `src/agentscope/app/storage/_model/_session.py`
  - `src/agentscope/app/storage/_model/_agent.py`
  - `src/agentscope/app/_service/_chat.py`
  - `src/agentscope/app/_service/_toolkit.py`
  - `src/agentscope/app/message_bus/_base.py`
  - `src/agentscope/agent/_agent.py`
