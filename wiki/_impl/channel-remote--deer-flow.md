---
title: "Channel & Remote Interface — DeerFlow"
category: L2
parent: "[[channel-remote]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 拥有同类项目中最完整的 channel 体系：Nginx 反向代理统一入口，三个后端服务各司其职，内置 Slack、Telegram、Feishu、WeCom 四个 IM 平台适配器，同时提供嵌入式 Python SDK（`DeerFlowClient`）和 ACP 协议用于跨 harness agent 互调。

## 架构分析

### 三服务 + Nginx 架构

Nginx 监听 `:2026`，作为统一入口反向代理到三个后端：

| 后端 | 端口 | 职责 |
|------|------|------|
| LangGraph Server | `:2024` | Agent runtime，处理 SSE 流式推理 |
| Gateway API (FastAPI) | `:8001` | REST API，IM channel 入站、suggestions、thread 管理 |
| Next.js Frontend | `:3000` | Web UI |

路由规则由 `backend/docker/nginx.conf` 定义，`/langgraph/` 前缀路由到 LangGraph Server，其余 API 路由到 Gateway，根路径路由到 Next.js。

### IM Channel 适配器

`backend/app/channels/` 下实现四个 IM 平台适配器，每个适配器包含：

1. **Webhook 入站处理**：接收平台推送的消息事件，验证签名
2. **消息解析**：将平台格式转换为统一的文本内容
3. **LangGraph 调用**：通过 `ChannelService` 转发到 agent runtime
4. **响应回写**：将 agent 输出通过平台 API 回送给用户

四个平台的适配器结构基本对称，差异主要在认证方式和 API client：

- **Slack**：Event API + OAuth Bot Token
- **Telegram**：Bot API + Webhook
- **Feishu (Lark)**：飞书开放平台 + 应用凭证
- **WeCom (企业微信)**：企业微信回调 + CorpID/Secret

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

- `backend/app/channels/` — Slack/Telegram/Feishu/WeCom 四个 IM 适配器目录
- `deerflow/client.py` — DeerFlowClient 嵌入式 SDK 实现
- `deerflow/tools/builtins/invoke_acp_agent_tool.py` — ACP 跨 harness 调用工具
- `backend/docker/nginx.conf` — Nginx 路由规则，三服务反向代理配置

## 设计亮点

- **最完整的 channel 覆盖**：4 个 IM 平台 + Web UI + Python SDK + ACP，覆盖了从开发者到企业用户的全场景接入需求
- **嵌入式 SDK 无服务器运行**：`DeerFlowClient` 允许零基础设施启动，极大降低集成门槛
- **ACP 协议实现跨 harness 互操作**：DeerFlow 可作为 meta-orchestrator 调度 Codex、Claude Code 等外部 agent，是同类项目中独有的能力
- **Nginx 统一入口**：三服务对外只暴露单一端口，简化防火墙配置和 SSL 证书管理

## 局限性

- **Channel 适配器偏薄**：当前适配器仅处理纯文本消息，不支持各平台的富交互元素（Slack Block Kit、Telegram InlineKeyboard 等）
- **SSE 单向推送**：使用 Server-Sent Events 而非 WebSocket，无法实现双向实时通信（客户端无法主动推送）
- **无 per-channel 权限模型**：不同 channel 的用户使用相同的 agent 权限，无法按渠道设置访问控制
- **IM 平台依赖平台侧 Webhook**：需要公网可达的 callback URL，本地开发需要 ngrok 等工具中转

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
