---
title: "Channel Remote — OpenHarness"
category: L2
parent: "[[channel-remote]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness 的"远程"能力体现在两个维度：Python 后端与 React/Ink TUI 前端通过 stdio JSON 协议通信的混合架构，以及通过 `BridgeSessionManager` 管理长生命周期子进程并支持定时触发执行的 cron 系统。当前"远程"本质上是本地子进程管理，尚未延伸至网络或云端。

## 架构分析

### Python + Node.js 混合架构

TUI 层由独立 Node.js 进程承载（React/Ink），与 Python 后端之间通过 stdio JSON 协议通信。Python 侧 `run_backend_host` 作为服务端接收来自 Node.js 的控制消息；`launch_react_tui` 负责 spawn Node.js 子进程并建立双向 stdio 管道。`ui/protocol.py` 定义消息格式，确保双方编解码一致。

这一设计避免了将整个项目迁移为 TypeScript monorepo 的成本，同时复用了 React/Ink 生态的终端 UI 能力。

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
- `openharness/services/cron.py` — cron 任务 JSON 持久化与调度逻辑
- `openharness/tools/remote_trigger_tool.py` — `RemoteTriggerTool`，`asyncio.create_subprocess_exec` 执行触发

## 设计亮点

- **混合语言架构**：Python 后端 + Node.js React/Ink 前端，stdio JSON 协议解耦，各语言专注擅长领域，无需强制统一技术栈
- **Cron 调度内置**：将定时 agent 执行能力纳入框架，支持自动化运维场景，JSON 存储简单可审计
- **WorkSecret 设计预留**：凭证编码机制为未来网络化扩展（多 agent 协作、远程任务分发）预留了接口形态

## 局限性

- **"远程"仅限本机**：无 WebSocket 服务端，无 OAuth，无云端 channel；所谓"远程"是本地子进程管理
- **WorkSecret 未激活**：SDK URL 和 WorkSecret 机制已设计但未连接任何活跃网络端点，属于未完成的架构预留
- **无多用户支持**：无用户身份系统，无权限隔离，无法支持多用户并发访问
- **stdio 协议脆弱性**：Python/Node.js 间 stdio JSON 通信在进程异常退出时缺乏健壮的重连和恢复机制

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
