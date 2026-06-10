---
title: "Channel Remote — OpenHarness"
category: L2
parent: "[[channel-remote]]"
source: openharness
source_version: "v0.1.9-51-g9b2efd7"
confidence: high
created: 2026-04-06
updated: 2026-06-09
---

## 概述

OpenHarness 的 channel/remote 能力需要分两层理解：

1. `src/openharness` core runtime：提供 engine、commands、memory、channels、tasks、bridge/session runner 等通用基础设施；
2. `ohmo/` 产品壳：个人 agent 应用，注入独立 workspace、personal memory、skills、plugins、gateway、session backend 和 Feishu group 等产品策略。

这条边界是 OpenHarness 当前最值得借鉴的地方：SDK core 不承载产品人格和个人 workspace；产品 agent shell 在外层组合 runtime、memory、channel 和 session。

## 架构分析

### Python + Node.js 混合架构

TUI 层由独立 Node.js 进程承载（React/Ink），与 Python 后端之间通过 stdio JSON 协议通信。Python 侧 `run_backend_host` 作为服务端接收来自 Node.js 的控制消息；`launch_react_tui` 负责 spawn Node.js 子进程并建立双向 stdio 管道。`ui/protocol.py` 定义消息格式，确保双方编解码一致。

这一设计避免了将整个项目迁移为 TypeScript monorepo 的成本，同时复用了 React/Ink 生态的终端 UI 能力。

### Core Channel Bus

OpenHarness core 已有 channel bus / manager 抽象：

- `src/openharness/channels/bus/events.py:8-24` — `InboundMessage` 统一 channel、sender/chat、content、media、metadata 和 session key；
- `events.py:27-36` — `OutboundMessage` 统一 channel/chat/content/reply/media/metadata；
- `src/openharness/channels/bus/queue.py:8-34` — `MessageBus` 用 inbound/outbound `asyncio.Queue` 解耦 channel 与 agent core；
- `src/openharness/channels/impl/manager.py:17-33` — `ChannelManager` 初始化 enabled channels 并持有 channels map；
- `manager.py:209-239` — `_dispatch_outbound()` 从 bus 消费 outbound message，再路由到对应 channel adapter。

当前 core bus 是进程内 queue，不是分布式 broker；它的价值是清晰拆开“平台 adapter”和“agent runtime”，不是直接解决多节点问题。

### ohmo 产品壳

`ohmo/gateway/runtime.py` 展示了产品层如何复用 OpenHarness runtime：

- `OhmoSessionRuntimePool` 按 chat/thread session key 缓存 `RuntimeBundle`；
- `get_bundle()` 优先恢复 `OhmoSessionBackend` 的 session snapshot，再调用 `build_runtime()`；
- 注入 `build_ohmo_system_prompt()`、ohmo private skills/plugins、personal memory backend；
- `include_project_memory=False`，避免个人 assistant 自动吸入项目 memory；
- `autodream_context` 指向 ohmo personal memory 的 memory/session 目录。

这说明 OpenHarness/ohmo 不是“一个框架里混入个人 agent 行为”，而是把产品行为放在壳层。

### BridgeSessionManager

`BridgeSessionManager` 管理长生命周期子进程（"bridge sessions"），捕获子进程的 stdout/stderr 输出。`WorkSecret` 和 SDK URL 将会话凭证编码为可传递的字符串，设计意图是支持将会话句柄交接给外部进程或工具，但当前未连接任何活跃的网络端点。

### Cron 调度系统

Cron 任务以 JSON 格式持久化于 `~/.openharness/crons.json`。`RemoteTriggerTool` 通过 `asyncio.create_subprocess_exec` 按需启动子进程执行预定义任务，支持定时触发 agent 执行场景（如周期性代码检查、定时报告生成）。

### 关键代码路径

- `openharness/bridge/session_runner.py` — bridge session 生命周期管理，子进程启动与输出捕获
- `openharness/bridge/manager.py` — `BridgeSessionManager`，多 bridge session 的集中管理
- `openharness/bridge/work_secret.py` — `WorkSecret` 凭证封装与 SDK URL 编码
- `openharness/ui/backend_host.py` — Python 侧 stdio JSON 服务端（`run_backend_host`）
- `openharness/ui/react_launcher.py` — Node.js 子进程 spawn 与 stdio 管道建立（`launch_react_tui`）
- `openharness/ui/protocol.py` — Python/Node.js 间 stdio 消息协议定义
- `openharness/channels/bus/events.py` — Inbound/Outbound message 契约
- `openharness/channels/bus/queue.py` — 进程内 async MessageBus
- `openharness/channels/impl/manager.py` — 多 channel 初始化与 outbound dispatch
- `ohmo/gateway/runtime.py` — 产品壳 session runtime pool、workspace/memory/skills/plugins 注入
- `openharness/services/cron.py` — cron 任务 JSON 持久化与调度逻辑
- `openharness/tools/remote_trigger_tool.py` — `RemoteTriggerTool`，`asyncio.create_subprocess_exec` 执行触发

## 设计亮点

- **混合语言架构**：Python 后端 + Node.js React/Ink 前端，stdio JSON 协议解耦，各语言专注擅长领域，无需强制统一技术栈
- **SDK core / product shell 分层**：OpenHarness core 保持通用；ohmo 在外层注入 personal workspace、private skills、gateway policy 和 persona
- **MessageBus 解耦 channel 和 agent core**：adapter 只负责平台输入输出，agent runtime 只消费统一消息模型
- **Cron 调度内置**：将定时 agent 执行能力纳入框架，支持自动化运维场景，JSON 存储简单可审计
- **WorkSecret 设计预留**：凭证编码机制为未来网络化扩展（多 agent 协作、远程任务分发）预留了接口形态

## 局限性

- **core MessageBus 是进程内 queue**：不能直接承担多节点 channel broker，需要替换为 Redis/Postgres/Kafka 等外部总线
- **产品能力不能直接等同于 SDK 能力**：ohmo 的 Feishu group、personal memory、private skills 是产品壳特性，不能写入 OpenHarness core 能力清单
- **WorkSecret 未激活**：SDK URL 和 WorkSecret 机制已设计但未连接任何活跃网络端点，属于未完成的架构预留
- **core 无完整多用户授权模型**：远程多用户授权主要在 ohmo/gateway 产品层处理
- **stdio 协议脆弱性**：Python/Node.js 间 stdio JSON 通信在进程异常退出时缺乏健壮的重连和恢复机制

## 来源

- 源码版本：`v0.1.9-51-g9b2efd7`
- 分析深度：源码级
