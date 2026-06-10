---
title: "multi-agent——agentscope"
category: L2
parent: "[[multi-agent]]"
source: "agentscope"
source_version: "v2.0.1-11-g0e5418e8"
concept: "multi-agent"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的多 agent 主线已经从旧版 `pipeline` / `MsgHub` / `A2AAgent`，转向**服务化 session + workspace + message bus + team tools**。当前源码里未找到旧页引用的 `src/agentscope/a2a/`、`pipeline/`、`realtime/`、`tts/` 路径；AgentCard / A2A / Nacos 的生产级注册发现能力应归到独立的 [[agent-registry-discovery--agentscope-java]]。

它现在更像一个可嵌入的多用户 Web/distributed agent runtime：FastAPI app factory 组合 storage、message bus、workspace manager、custom subagent templates 和 custom agent class；`ChatService` 通过 `MessageBus` 做 session 级分布式锁、event replay、live fan-out；team worker 由 `AgentCreate` 创建并继承 leader workspace/model，任务通过 inbox + wakeup 投递。

---

## 当前架构

### App Factory 依赖注入

`create_app()` 是 AgentScope 2.x 的服务化入口，它不直接绑定具体存储或消息系统，而是接收外部注入：

- `storage: StorageBase`
- `message_bus: MessageBus`
- `workspace_manager: WorkspaceManagerBase`
- `extra_agent_middlewares`
- `extra_agent_tools`
- `custom_subagent_templates`
- `custom_agent_cls`

这些对象被写入 `app.state`，再由 routers/services 通过依赖注入读取。这个结构把 Web API、运行时状态、workspace、team worker 模板拆开，使同一个 runtime 可以部署为本地 Web app、私有服务或更大的产品 shell。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_app.py:32` — `create_app()` 参数
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_app.py:138` — runtime 依赖写入 `app.state`

### ChatService 是统一运行路径

`ChatService.run()` 是 HTTP chat endpoint 和 wakeup dispatcher 的共同入口。它按固定顺序：

1. 读取 `AgentRecord` 与 `SessionRecord`
2. 根据 `SessionConfig.workspace_id` 解析 workspace
3. 把 workspace workdir 注入 `PermissionContext`
4. 组装 toolkit、middlewares、model/fallback model
5. 使用 `AgentState` 构建 agent
6. 在 `message_bus.session_run(session_id)` 的分布式锁内执行 `reply_stream()`
7. 将 agent events 发布到 session event stream
8. 持久化 reply message 和更新后的 session state

这意味着多 agent / Web session / wakeup 触发不是三套逻辑，而是共享同一条 run pipeline。对 agent-os 来说，这个结构适合抽象成：

- `SessionStore`
- `WorkspaceManager`
- `MessageBus`
- `RunController`
- `AgentAssembler`

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:51` — `ChatService` 说明 MessageBus 负责 distributed lock、event replay、live fan-out
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:172` — 加载 agent/session 并解析 workspace
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:222` — workspace workdir 注入 permission context
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:354` — `async with self._message_bus.session_run(session_id)`

### Team Tools 驱动多 Agent

AgentScope 2.x 的多 agent 不是 peer-to-peer mesh，而是 leader session 通过工具管理 team：

- 普通 user-owned agent 获得 `TeamCreate` / `AgentCreate` / `TeamSay` / `TeamDelete`
- worker agent（`agent_record.source == "team"`）只获得 `TeamSay`
- `AgentCreate` 会创建隐藏的 worker `AgentRecord` 和 worker `SessionRecord`
- worker session 继承 leader session 的 `workspace_id`、chat model、fallback model
- 初始任务被包装成 `HintBlock`，写入 worker inbox，并调用 `enqueue_wakeup()` 唤醒 worker session

这个设计把“创建子 agent”落到 storage/session 模型，而不是只在内存里生成一个临时对象。好处是 worker 可被 Web session 生命周期、message bus、permission context 和 workspace 统一管理；代价是协作拓扑主要是 leader-driven。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_toolkit.py:142` — worker 只拿 `TeamSay`，leader 拿全量 team toolset
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_types.py:29` — `SubAgentTemplate` 作为可注册 worker 蓝图
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_tools/_agent_create.py:349` — 创建 `source="team"` 的 worker agent
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_tools/_agent_create.py:376` — worker session 继承 leader workspace/model
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_tools/_agent_create.py:412` — inbox push + wakeup 投递初始任务

### MessageBus 承担协作基础设施

MessageBus 同时承载三类职责：

- session 级互斥：避免同一 session 在多个进程并发执行
- event replay/live fan-out：让前端或调用方从 SSE 订阅 agent events
- inbox/wakeup：team worker、外部执行结果、后台任务完成后唤醒 idle session

Redis 实现通过 `SET NX EX` 获取分布式锁，并用 heartbeat 续租；释放时校验 token，避免误删其他持有者的锁。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/message_bus/_redis_message_bus.py:449` — Redis distributed lock
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_router/_session.py:421` — session SSE endpoint
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_router/_session.py:472` — SSE 先 replay buffered events，再 live subscribe

### Workspace / Permission 是安全边界

AgentScope 2.x 把 workspace 作为工具和权限边界：

- `LocalWorkspace` 直接暴露 Bash/Edit/Glob/Grep/Read/Write 等本地工具
- `DockerWorkspace` 通过容器内 MCP gateway 暴露工具
- `E2BWorkspace` 对接远程沙箱
- `ChatService` 会把当前 workspace workdir 写入 session permission context

注意：Local workspace manager 当前未用 `user_id` 参与 workdir 隔离，多用户 Web 形态若使用 Local workspace，需要外层再做租户隔离。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/workspace/_base.py:11` — Workspace 抽象
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/workspace/_local_workspace.py:670` — Local workspace 暴露本地工具
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/workspace/_docker/_docker_workspace.py:410` — Docker workspace 走 MCP gateway
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/workspace_manager/_local_workspace_manager.py:116`、`:136` — local workdir 由 `os.path.join(basedir, agent_id)` 组成，`user_id/session_id` 仅为接口兼容参数

---

## 设计亮点

### 1. 多 agent 复用 session/run 基础设施

worker 不只是内存对象，而是有独立 `AgentRecord`、`SessionRecord`、`AgentState` 和 inbox 的隐藏 agent。这样 team worker 可以天然复用 session lock、event stream、workspace、permission、scheduler/background task 这些 Web runtime 能力。

### 2. SubAgentTemplate 把 worker 类型做成数据蓝图

`SubAgentTemplate` 把 type、description、system prompt template、context config、ReAct config、permission context、tasks context 做成可序列化配置。leader 调用 `AgentCreate` 时只需要选择 `subagent_type`，worker 的能力边界由模板决定。

### 3. Inbox + wakeup 适合分布式 idle session

team message 不直接同步调用 worker，而是写 inbox 后 enqueue wakeup。这样 worker 可以在另一个进程中运行，也可以在当前进程空闲时被 dispatcher 唤醒。它牺牲了即时 peer-to-peer 反馈，但换来了可恢复、可排队、可跨进程的执行模型。

### 4. MessageBus 将锁、事件和唤醒统一抽象

把 distributed lock、event replay/live fan-out、inbox/wakeup 都放进 MessageBus，能避免 ChatService 直接依赖 Redis 或 SSE 细节。agent-os 如果要同时支持本地代理和 Web 分布式 agent，可以定义同一套 `MessageBus` 抽象，再分别实现 in-process/local-file/Redis/NATS 版本。

### 5. Workspace 与 PermissionContext 挂到 session

worker 继承 leader workspace，且 ChatService 每轮把 workspace workdir 写进 permission context。这个路径保证“这个 agent 能操作什么文件/工具”由 session workspace 决定，而不是每个 tool 自己临时判断。

---

## 局限性

### 1. 当前 Python 2.x 不是去中心化 agent mesh

team worker 由 leader tool 创建和唤醒；worker 可通过 `TeamSay` 回报，但整体仍是 leader-driven。它不是多个 agent 对等协商的 actor mesh，也没有展示复杂拓扑路由。

### 2. Local workspace 多用户隔离偏弱

Local workspace manager 的 workdir 当前只由 `basedir/agent_id` 组成，`user_id/session_id` 未参与本地路径。若 Web 多租户直接使用 Local workspace，需要额外按租户隔离 root、权限校验和清理策略。

### 3. MessageBus 抽象偏运行时核心，部署仍要选后端

内存 MessageBus 适合本地/单进程；Redis MessageBus 适合多进程 Web 服务。agent-os 需要把这层做成明确后端矩阵，否则同一套 API 在本地和分布式形态下的可靠性差异会被隐藏。

### 4. Python 2.x 不再承载 AgentCard/A2A/Nacos 事实

旧版 `A2AAgent` / AgentCard resolver / Nacos resolver 不在当前 Python 2.x 源码中。若要借鉴 AgentCard、A2A server、Nacos registry、Spring Boot auto-configuration，应使用 [[agent-registry-discovery--agentscope-java]]。

---

## 对 agent-os 的借鉴

P0：把 agent runtime 切成 ABC，再做本地/分布式实现：

- `SessionStore`: local sqlite/json vs server DB
- `MessageBus`: in-process queue vs Redis/NATS
- `WorkspaceManager`: local filesystem vs Docker/E2B/remote workspace
- `PermissionContext`: local allowlist vs tenant policy
- `RunController`: direct function call vs Web run lock + event stream
- `AgentAssembler`: 从 session state + workspace + tools + memory 组装 agent

P1：Team worker 不要只做内存 subagent。至少在 Web/distributed 形态下，worker 应有独立 session/state/inbox，这样才能被恢复、审计、限权和跨进程调度。

P1：SubAgentTemplate 应作为序列化配置存在。不要把“researcher/coder/reviewer worker 类型”写死在 prompt 或代码分支里，应由模板声明 system prompt、context、tool/permission 范围和初始 task context。

P2：本地代理形态可以用 in-process MessageBus 和 local workspace 快速实现，但接口上不要偷懒跳过 MessageBus/WorkspaceManager，否则未来切到 Web 分布式形态会重写 agent loop。

---

## 来源

- 源码版本：`v2.0.1-11-g0e5418e8`
- 分析深度：源码级
- 核心文件：
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_app.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_toolkit.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_tools/_agent_create.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_types.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/message_bus/_redis_message_bus.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_router/_session.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/workspace/`
