---
title: "session-recovery——agentscope"
category: L2
parent: "[[session-recovery]]"
source: "agentscope"
source_version: "v2.0.1-11-g0e5418e8"
concept: "session-recovery"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的 session recovery 已从旧版 `StateModule + SessionBase` 迁移到服务化 runtime：`StorageBase` 持久化 `SessionRecord.state: AgentState`、`ChatService` 每轮从 storage 重建 agent、`MessageBus.session_run()` 用 session 级分布式锁串行化执行并提供事件 replay/live fan-out。

当前源码中未找到旧页引用的 `src/agentscope/session/`、`StateModule`、`SessionBase`、`JSONSession`、`RedisSession`、`TablestoreSession` 路径。旧的“多 agent state_modules_mapping 批量 save/load”结论不再代表 2.x。

---

## 架构分析

### SessionRecord

`SessionRecord` 当前是存储中的恢复单元：

| 字段 | 作用 |
|---|---|
| `user_id` / `agent_id` | 租户和 agent 归属 |
| `source` / `source_schedule_id` | 用户创建或 schedule 创建 |
| `team_id` | team worker / leader 所属团队 |
| `config` | workspace、session name、chat model、fallback model |
| `state` | `AgentState`，包含 summary、context、reply_id、cur_iter、permission/tool/task context |

### StorageBase

`StorageBase` 抽象了 session 的生命周期：

- `upsert_session(user_id, agent_id, config, state, session_id, source, source_schedule_id)`
- `get_session(user_id, agent_id, session_id)`
- `update_session_state(user_id, agent_id, session_id, state)`
- `list_sessions()` / `delete_session()`
- `upsert_message()` / `get_message()` 等消息持久化接口

热路径是 `update_session_state()`：每轮 chat 结束后只更新 mutable state。

### ChatService 恢复路径

`ChatService.run()` 是 HTTP chat endpoint 和 wakeup dispatcher 的共同入口。每次运行：

1. 从 storage 读取 `AgentRecord` 和 `SessionRecord`
2. 用 `SessionConfig.workspace_id` 解析 workspace
3. 将 workspace workdir 写入 `state.permission_context`
4. 组装 toolkit、middlewares、model/fallback model
5. 用 `session_record.state` 创建 `Agent`
6. 在 `message_bus.session_run(session_id)` 锁内执行 `agent.reply_stream()`
7. 将事件发布到 message bus
8. upsert reply message
9. 调用 `storage.update_session_state()` 保存最新 `AgentState`

这是一种“每轮重建 agent + 持久化 state”的服务端恢复模型，而不是长生命周期对象快照。

### MessageBus 运行控制

`MessageBus` 提供：

- `session_run(session_id)`：session 级互斥锁，防止多进程并发运行同一 session
- `session_publish_event()`：写 replay log 并 live publish
- `session_read_events()` / `session_subscribe_events()`：SSE replay + live fan-out
- `session_publish_cancel()`：跨进程 cancel broadcast
- `inbox_push()` / `inbox_drain()` / `enqueue_wakeup()`：team/background/scheduler 的 idle session 唤醒

锁退出前会 trim replay log；完整 reply message 已写入 storage 后，事件 log 不再承担长期存储职责。

---

## 关键代码路径

- `src/agentscope/app/storage/_base.py` — `StorageBase`
- `src/agentscope/app/storage/_model/_session.py` — `SessionRecord` / `SessionConfig`
- `src/agentscope/state/_state.py` — `AgentState`
- `src/agentscope/app/_service/_chat.py` — `ChatService.run()` / `_run_impl()`
- `src/agentscope/app/message_bus/_base.py` — session lock、event replay、cancel、inbox、wakeup
- `src/agentscope/app/storage/_redis_storage.py` — Redis storage 实现
- `src/agentscope/app/_router/_session.py` — session REST/SSE API

---

## 设计亮点

### 1. 每轮重建 agent，state 才是恢复边界

Agent 对象本身不需要跨请求存活。Web runtime 每轮从 storage + workspace + tools + model config 重新装配 agent，只把 `AgentState` 作为可恢复状态。这非常适合多进程服务。

### 2. Session lock 和 state 写入在同一运行边界

`ChatService` 在 `message_bus.session_run(session_id)` 锁内执行 agent，并在释放锁前 `update_session_state()`。这避免另一个进程拿到锁后读到旧 state。

### 3. Replay log 与持久消息分工清晰

message bus replay log 只服务 in-flight SSE 订阅，完整消息历史由 storage 的 message record 承担。这样事件流可以短期 trim，长期存储保持结构化消息。

### 4. Continuation event 支持半暂停工具调用

当工具需要用户确认或外部执行结果时，`reply_id` 和上下文保存在 `AgentState` 中。下一次 `UserConfirmResultEvent` / `ExternalExecutionResultEvent` 到来时，agent 可从等待中的 tool call 继续。

### 5. Inbox + wakeup 支持 idle session 恢复

team message、background task 完成、schedule trigger 都可以先写 inbox，再 enqueue wakeup。session 不需要常驻进程中，也能被事件重新拉起。

---

## 局限性

### 1. 恢复粒度是 run/turn 边界

状态在 chat run 结束后写回。进程在工具执行中途崩溃时，已发布但未持久化成 reply message/state 的部分仍可能丢失或需要上层 reconcile。

### 2. MessageBus replay log 不是长期审计日志

`session_run()` 退出时会 trim replay log。需要审计或完整 replay 的系统必须依赖 storage message / external trace，而不是 message bus stream。

### 3. StorageBase 是抽象，具体一致性取决于后端

当前抽象允许 Redis storage 等实现，但事务、TTL、跨 key 原子性、消息与 state 的一致性需要看后端实现和部署参数。

### 4. Local workspace 恢复隔离有限

Local workspace manager 接口接受 `user_id/session_id`，但当前 workdir 只由 `basedir/agent_id` 组成，`user_id` 和 `session_id` 被丢弃。多租户 Web 直接使用 Local workspace 需要外层隔离 root 和权限策略。

### 5. AgentState schema 演进需要显式兼容

`AgentState` 是 Pydantic 模型，旧 state 的字段演进仍需 migration 或默认值策略；这部分没有独立版本迁移层。

---

## 对 agent-os 的借鉴

agent-os 同时支持本地代理和 Web 分布式代理时，应拆出这些 ABC：

- `SessionStore`: 保存 `SessionRecord`、message、state
- `RunController`: 本地 direct run vs 分布式 lock + cancel
- `EventBus`: local callback/SSE/Redis/NATS
- `AgentAssembler`: 从 session config、state、workspace、tools 组装 agent
- `ContinuationManager`: 处理用户确认、外部执行结果和等待中的 tool call

本地形态可以用 SQLite/JSON + in-process lock；Web 形态需要分布式锁、event replay 和 wakeup queue。

---

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
- 核心文件：
  - `src/agentscope/app/storage/_base.py`
  - `src/agentscope/app/storage/_model/_session.py`
  - `src/agentscope/state/_state.py`
  - `src/agentscope/app/_service/_chat.py`
  - `src/agentscope/app/message_bus/_base.py`
  - `src/agentscope/app/storage/_redis_storage.py`
