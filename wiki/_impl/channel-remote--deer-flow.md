---
title: "Channel & Remote Interface — DeerFlow"
category: L2
parent: "[[channel-remote]]"
source: deer-flow
source_version: "v2.0-m1-rc2-11-g16391e35"
confidence: high
created: 2026-04-07
updated: 2026-06-10
---

## 概述

DeerFlow 的 channel 体系已经从“四个薄 IM adapter”演进为中心化 `ChannelService` / `ChannelManager` / `MessageBus`：七类 channel registry、topic 到 thread 映射、channel/user session override、skill whitelist、入站上传、出站 artifact attachment，以及 Feishu/WeCom streaming。当前生产 compose 是 Nginx + Frontend + Gateway（可选 provisioner）：Gateway 内嵌 LangGraph-compatible API/runtime，nginx 将 `/api/langgraph/*` 重写到 Gateway，而不是再运行独立 LangGraph Server 容器。

## 架构分析

### Nginx + Gateway 内嵌 runtime 架构

Nginx 监听 `:2026`，作为统一入口反向代理到 frontend 与 gateway：

| 后端 | 端口 | 职责 |
|------|------|------|
| Gateway API (FastAPI) | `:8001` | REST API、LangGraph-compatible API/runtime、IM channel 入站、suggestions、thread 管理 |
| Next.js Frontend | `:3000` | Web UI |
| Provisioner | 可选 | Kubernetes sandbox provisioner |

路由规则由 `docker/nginx/nginx.conf` 定义。`/api/langgraph/*` 会先 rewrite 为 `/api/*`，再 proxy 到 Gateway；其余 API 同样进 Gateway，根路径路由到 Next.js。

### IM Channel 适配器与 ChannelManager

`backend/app/channels/` 下由 `ChannelService` 注册 channel，并由 `ChannelManager` 统一管理入站、出站、session merge 和附件安全：

1. **Webhook 入站处理**：接收平台推送的消息事件，验证签名
2. **消息解析**：将平台格式转换为统一的文本内容
3. **LangGraph 调用**：通过 `ChannelService` 转发到 agent runtime
4. **响应回写**：将 agent 输出通过平台 API 回送给用户

最新源码中 channel registry 覆盖七类 channel：`dingtalk`、`discord`、`feishu`、`slack`、`telegram`、`wechat`、`wecom`。四个平台适配器之外还包含更完整的 MessageBus / manager 组合：

- `backend/app/channels/service.py:20-28`、`:55-77` — 七渠道注册与生命周期
- `backend/app/channels/message_bus.py:32-60`、`:64-110` — inbound files/topic 与 outbound attachments
- `backend/app/channels/manager.py:54-62` — channel capabilities
- `backend/app/channels/manager.py:469-516` — outputs-only attachment 安全边界
- `backend/app/channels/manager.py:546-651` — inbound upload
- `backend/app/channels/manager.py:700-769` — session layer merge、channel-user override、user-scoped run context
- `backend/app/channels/manager.py:992-1172` — skill whitelist、stream/chat/commands

边界：这不是完整通用授权系统；它提供的是 per-channel/per-user session、agent、skill/context 配置，以及 artifact 输入输出边界。

### 嵌入式 Python SDK

`deerflow/client.py` 提供 `DeerFlowClient`，允许在不启动任何服务器的情况下直接在 Python 进程内调用 DeerFlow agent：

```python
client = DeerFlowClient()
response = client.chat("Research the latest AI trends")
```

`DeerFlowClient` 直接实例化 LangGraph graph 对象，跳过 HTTP 层，适合 notebook、脚本、集成测试等场景。

### ACP 协议跨 Harness 互调

`deerflow/tools/builtins/invoke_acp_agent_tool.py` 实现 ACP (Agent Communication Protocol) 工具，使 DeerFlow 可以将外部 agent（Codex、Claude Code 等）作为子 agent 调用：

- 工具接收 `agent_url` 和 `task` 参数
- 通过 ACP 协议发送任务请求，等待结果
- 结果作为工具调用返回值注入 DeerFlow 的 agent context

这使得 DeerFlow 可以作为 orchestrator，协调多个异构 agent 系统。

### 关键代码路径

- `backend/app/channels/` — DingTalk/Discord/Feishu/Slack/Telegram/WeChat/WeCom 七个 channel 适配器
- `backend/app/channels/service.py` — ChannelService 注册与生命周期
- `backend/app/channels/manager.py` — ChannelManager，session merge / attachments / upload / slash command / skill whitelist
- `backend/app/channels/message_bus.py` — inbound topic/files 与 outbound attachments
- `deerflow/client.py` — DeerFlowClient 嵌入式 SDK 实现
- `deerflow/tools/builtins/invoke_acp_agent_tool.py` — ACP 跨 harness 调用工具
- `docker/docker-compose.yaml` — Nginx/Frontend/Gateway/可选 provisioner 生产 compose
- `docker/nginx/nginx.conf` — Nginx 路由规则，`/api/langgraph/*` rewrite 到 Gateway

## 设计亮点

- **中心化 channel manager**：多平台差异被收敛到 ChannelManager / MessageBus，channel 层能处理 session override、skill whitelist、入站上传和出站 artifact
- **嵌入式 SDK 无服务器运行**：`DeerFlowClient` 允许零基础设施启动，极大降低集成门槛
- **ACP 协议实现跨 harness 互操作**：DeerFlow 可作为 meta-orchestrator 调度 Codex、Claude Code 等外部 agent，是同类项目中独有的能力
- **Nginx 统一入口**：frontend/gateway 对外只暴露单一端口，简化防火墙配置和 SSL 证书管理；LangGraph-compatible API 不再需要单独服务端口

## 局限性

- **仍非通用权限系统**：channel 层有 session/agent/skill/context 配置，但不是完整 RBAC/ABAC 权限模型
- **SSE 单向推送**：使用 Server-Sent Events 而非 WebSocket，无法实现双向实时通信（客户端无法主动推送）
- **不是完整 RBAC/ABAC**：部分 channel 已有轻量访问限制（Slack/Telegram `allowed_users`、Discord `allowed_guilds`/`allowed_channels`），ChannelManager 也支持 session override；但这还不是跨渠道统一的角色/属性权限模型，仍需和 workspace/sandbox policy 联动
- **IM 平台依赖平台侧 Webhook**：需要公网可达的 callback URL，本地开发需要 ngrok 等工具中转

## 来源

- 源码版本：`v2.0-m1-rc2-11-g16391e35`
- 分析深度：源码级
