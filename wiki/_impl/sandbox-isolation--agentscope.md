---
title: "Sandbox Isolation — AgentScope Python"
category: L2
parent: "[[sandbox-isolation]]"
source: agentscope
source_version: "v2.0.1-11-g0e5418e8"
confidence: high
created: 2026-06-09
updated: 2026-06-09
relations:
  - target: "[[sandbox-isolation]]"
    type: implements
  - target: "[[channel-remote]]"
    type: depends_on
  - target: "[[tool-system]]"
    type: feeds
---

## 概述

AgentScope Python 2.x 把 workspace 作为工具执行与权限判断的核心边界。它不是只给 agent 一个工作目录，而是按 workspace 类型决定工具如何暴露：Local 直接暴露本地文件/命令工具；Docker 通过容器内 MCP gateway 执行；E2B 提供远程沙箱形态。`PermissionEngine` 再根据 permission mode 和 workspace workdir 做运行时审批。

## 架构

| Workspace | 工具暴露方式 | 隔离含义 |
|---|---|---|
| Local | 直接暴露 Bash/Edit/Glob/Grep/Read/Write | 适合本地代理；隔离依赖本机目录和权限配置 |
| Docker | 不直接暴露内置工具，工具经容器内 MCP gateway | 容器边界 + bearer token gateway |
| E2B | 远程沙箱 workspace | 适合托管执行和更强隔离需求 |

关键代码：

- `src/agentscope/workspace/_base.py:11` — WorkspaceBase 明确 Local / Docker / E2B 三类实现
- `src/agentscope/workspace/_local_workspace.py:670` — Local workspace 暴露 Bash/Edit/Glob/Grep/Read/Write
- `src/agentscope/workspace/_docker/_docker_workspace.py:410` — Docker workspace 通过容器内 MCP gateway 暴露工具
- `src/agentscope/workspace/_mcp_gateway/_mcp_gateway_app.py:26` — gateway 除 `/health` 外要求 bearer token
- `src/agentscope/permission/_types.py:18` — PermissionMode：DEFAULT / ACCEPT_EDITS / EXPLORE / BYPASS / DONT_ASK
- `src/agentscope/permission/_engine.py:76` — PermissionEngine 按 mode dispatch
- `src/agentscope/app/_service/_chat.py:222` — workspace workdir 注入 permission context

## 设计亮点

### 1. Workspace 是权限边界，不只是路径参数

ChatService 会把 workspace workdir 注入 permission context。这使文件编辑、命令执行、读写工具都能共享同一个 workspace root，而不是每个工具自行判断“当前项目目录”。

### 2. Docker workspace 把工具协议移入容器

Docker workspace 不把本地内置工具直接暴露给 agent，而是通过容器内 MCP gateway 统一执行。这让工具调用协议、身份 token 和容器边界绑定，适合从 local agent 过渡到 Web/distributed agent。

### 3. Permission mode 与 workspace 组合

PermissionMode 区分默认、自动接受编辑、探索、绕过、不询问等模式。它不是沙箱本身，但决定 workspace 内工具调用是否需要审批。

## 局限

- Local workspace manager 当前未用 `user_id` 参与 workdir 隔离：`src/agentscope/app/workspace_manager/_local_workspace_manager.py:80`。多用户 local profile 仍需要额外目录/权限隔离。
- Docker gateway 的 bearer token 保护是进程/容器内协议边界，不等同于完整多租户授权系统。
- E2B 路径提供更强隔离，但成本、依赖和可用性取决于外部服务。

## 对 agent-os 的借鉴

agent-os 同时支持 local proxy agent 和 Web distributed agent 时，应把 workspace 设计为一等抽象：

- `LocalWorkspace`：本地文件系统 + 本机权限审批
- `ContainerWorkspace`：容器/沙箱 + gateway tool protocol
- `RemoteWorkspace`：托管沙箱或远程执行环境

每个 workspace 都应能提供 workdir、tool gateway、permission context 和 evidence URI policy，而不是把路径散落在工具参数里。

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
