---
title: "channel-remote——agentscope"
category: L2
parent: "[[channel-remote]]"
source: agentscope
source_version: "v2.0.1-11-g0e5418e8"
concept: channel-remote
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的 channel/remote 主线已经不是旧版 `realtime/` + `tts/`。当前源码中未找到旧页引用的 `realtime/`、`tts/` 路径；真实主线是 **FastAPI app factory + session REST/SSE + MessageBus event replay/live fan-out + Redis session lock**。

这让 AgentScope 更接近 Web/distributed agent backend：HTTP endpoint 只负责接入，真正的运行路径由 `ChatService`、`MessageBus`、`StorageBase`、`WorkspaceManagerBase`、`AgentState` 串起来。跨系统 AgentCard/A2A/Nacos 能力应看独立的 [[agent-registry-discovery--agentscope-java]]。

---

## 当前架构

### FastAPI App Factory

`create_app()` 是可嵌入的服务入口。调用方必须注入：

- `StorageBase`
- `MessageBus`
- `WorkspaceManagerBase`

还可以注入：

- `extra_agent_middlewares`
- `extra_agent_tools`
- `custom_subagent_templates`
- `custom_agent_cls`

这使 AgentScope 的 remote runtime 可以挂载到已有 FastAPI 应用，或作为独立服务运行；Web 层不直接创建 agent，而是把依赖放进 `app.state` 供 router/service 使用。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_app.py:32` — `create_app()` 参数
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_app.py:138` — storage/message_bus/workspace_manager 写入 app state

### Session REST + SSE

Session router 提供消息查询和事件订阅。SSE endpoint 的关键路径是：

1. 校验当前 `user_id` / `agent_id` 是否拥有 session
2. 读取 `message_bus.session_read_events(session_id)`，先 replay 当前 run 的 buffered events
3. 启动 feeder task 订阅 `message_bus.session_subscribe_events(session_id)`
4. 主循环从 queue 取 live events
5. 每 30 秒输出一次 SSE heartbeat comment frame

这个 replay-then-live 模式适合浏览器刷新、移动网络断线或前端晚订阅的场景：只要 replay log 还在，客户端不会错过当前 run 已经产生的事件。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_router/_session.py:421` — `/{session_id}/stream` SSE endpoint
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_router/_session.py:472` — replay buffered events
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_router/_session.py:476` — live subscribe feeder task
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_router/_session.py:500` — 30 秒 heartbeat

### ChatService 统一 run lifecycle

`ChatService` 是 HTTP chat endpoint 和 wakeup dispatcher 的共同执行路径。它在 `MessageBus.session_run(session_id)` 内运行 agent，保证同一 session 只有一个 run 在执行；run 中产生的每个 event 都通过 `session_publish_event()` 写入 replay log 并推送 live subscribers。

这个设计把 channel 层压薄：REST/SSE 不需要知道 agent loop 细节，只需要读写 session、提交输入、订阅事件。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:51` — MessageBus 提供 distributed lock、event replay、live fan-out
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:354` — `async with self._message_bus.session_run(session_id)`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:372` — reply events 发布到 session event stream
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:416` — 更新后的 session state 在 lock 内持久化

### Redis MessageBus 分布式锁

Redis MessageBus 的锁实现使用：

- `SET key token NX EX ttl_secs` 原子抢锁
- heartbeat task 按 `ttl_secs / 2` 续租
- 释放时先 `GET` 校验 token，再 `DEL`

这对 Web/distributed agent 很关键：没有 session lock，同一个 session 可能被 HTTP 请求、wakeup、外部执行结果同时触发，导致 agent state 和 reply message 被并发写坏。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/message_bus/_redis_message_bus.py:449` — `acquire_lock()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/message_bus/_redis_message_bus.py:479` — `SET NX EX`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/message_bus/_redis_message_bus.py:485` — heartbeat
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/message_bus/_redis_message_bus.py:503` — token guarded release

### Workspace/Permission 与 channel 绑定

每轮 run 会根据 session 的 `workspace_id` 解析 workspace，并把 workspace workdir 写入 permission context。这样从 HTTP channel 进入的用户消息不会绕开 workspace 权限边界；工具能力由 session workspace 和 toolkit assembly 决定。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:172` — 解析 session workspace
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:222` — workspace workdir 注入 permission context
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_toolkit.py:93` — workspace tools 进入 toolkit
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_toolkit.py:194` — workspace skills/MCPs 进入 toolkit

---

## 设计亮点

### 1. Replay-first SSE

SSE 不只是 live pub/sub。连接建立后先 replay 当前 run 的 buffered events，再进入 live subscribe。这比纯 live SSE 更适合 agent run，因为前端经常晚于 run 启动才接上事件流。

### 2. Channel 层不承载 agent 业务逻辑

Router 做鉴权、参数解析和 streaming response；真正 run lifecycle 在 `ChatService`。这让 HTTP、wakeup、外部执行结果恢复能共享同一条状态更新路径。

### 3. Session lock 在 MessageBus 层

把 lock 放进 MessageBus，而不是散落在 router 或 storage adapter 中，可以让 run lifecycle 统一使用同一个互斥语义。不同部署只需要替换 MessageBus 后端。

### 4. App Factory 支持产品壳扩展

`extra_agent_middlewares`、`extra_agent_tools`、`custom_subagent_templates`、`custom_agent_cls` 给产品层留了扩展点，但不污染 AgentScope core。agent-os 可以借鉴这个做法，把 local proxy agent 和 Web distributed agent 做成不同 shell。

### 5. Wakeup / inbox 支持异步协作

team message、background task completion、external execution result 都可以转成 inbox/wakeup，而不是要求同步调用同一个 agent 实例。这是从本地 agent 升级到 Web 分布式 agent 的关键抽象。

---

## 局限性

### 1. SSE 是单向通道

SSE 适合事件下行，但用户输入、确认、外部执行结果仍要走 REST。若 agent-os 需要双向实时控制、语音/低延迟交互或复杂前端协作，后续仍可能需要 WebSocket/WebRTC。

### 2. Replay log 生命周期需要产品层明确

当前 L2 只确认 SSE 会先 replay buffered events；具体 replay log 的保留策略、trim 时机、失败恢复窗口需要部署时明确，否则前端断线过久仍可能丢事件。

### 3. Local workspace 不等于多租户沙箱

AgentScope 提供 Local/Docker/E2B workspace 抽象，但 Local workspace 对 Web 多用户并不是安全沙箱。对外暴露 channel 时，应优先使用 Docker/E2B 或外层租户隔离。

### 4. Python 2.x 不再是旧 Realtime/TTS 参考

旧 `realtime/`、`tts/` 目录不在当前 Python 2.x 源码中。多模态实时 API、TTS、旧 ChatRoom 相关内容不应继续作为本页 source-backed 结论。

---

## 对 agent-os 的借鉴

P0：为 Web distributed agent 定义 `RunController`，把 run lock、event publish、state persistence 放在同一条事务边界内。不要让 HTTP handler 直接调用 agent loop。

P0：`MessageBus` 应同时支持 event stream 和 session wakeup。local 形态可以用 in-process queue；Web 形态可用 Redis/NATS；接口保持一致。

P1：channel event stream 采用 replay-first。前端刷新后先补当前 run 已发事件，再接 live stream，避免“run 已开始但 UI 订阅晚了”的空洞。

P1：REST/SSE 可作为第一版 Web channel。等到需要双向低延迟控制时，再升级 WebSocket；不要一开始把 channel 和 agent loop 绑死在 WebSocket 生命周期上。

P1：Workspace/Permission 必须绑定 session。所有从 channel 进入的工具调用都应由 session workspace 统一决定权限，而不是由 adapter 自己决定。

---

## 来源

- 源码版本：`v2.0.1-11-g0e5418e8`
- 分析深度：源码级
- 核心文件：
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_app.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_router/_session.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_toolkit.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/message_bus/_redis_message_bus.py`
