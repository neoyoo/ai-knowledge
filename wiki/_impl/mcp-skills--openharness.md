---
title: "MCP & Skills — OpenHarness"
category: L2
parent: "[[mcp-skills]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness 的 MCP 集成通过 Python `mcp` SDK 连接 stdio 服务器，每个 MCP 工具被包装为 `McpToolAdapter`，命名规则为 `mcp__servername__toolname`。Skills 是带可选 YAML frontmatter 的 `.md` 文件，通过 `skill(name=...)` 工具调用时直接返回完整 markdown 内容。两者通过统一的 `plugin.json` 格式集成，该格式兼容 Claude Code 插件生态，支持 `${CLAUDE_PLUGIN_ROOT}` 变量替换。

## 架构分析

### MCP 客户端架构

`McpClientManager` 使用 `AsyncExitStack` 管理多个 stdio 服务器的生命周期，连接时调用 `session.list_tools()` 和 `session.list_resources()` 枚举能力。每个发现的工具封装为 `McpToolAdapter`，统一注入到工具注册表中。命名前缀 `mcp__` 使模型可以区分原生工具和 MCP 工具。

### Skills 加载与索引

Skills 为纯 `.md` 文件，支持 YAML frontmatter 声明 `name` 和 `description`。加载顺序（优先级从低到高）：

1. 包内置 skills 目录
2. `~/.openharness/skills/`（用户全局）
3. 插件目录中声明的 skills

`SkillRegistry` 将所有已加载 skills 的名称和描述索引到 system prompt 中，模型调用 `skill(name="...")` 时，`skills/loader.py` 读取对应 `.md` 文件的完整内容作为工具返回值，模型即可按内容执行。

### 统一插件格式

`plugin.json` manifest 是核心集成点，单个插件可同时声明：

- `skills`：`.md` 文件路径列表
- `commands`：自定义斜杠命令
- `hooks`：hook 定义（对接 hooks 子系统）
- `mcp`：MCP 服务器配置

路径中支持 `${CLAUDE_PLUGIN_ROOT}` 变量，解析为插件根目录的绝对路径，实现可移植的路径引用。该格式与 Claude Code 的 `.claude-plugin/plugin.json` 规范兼容，可直接复用 Claude Code 生态中的插件。

### 关键代码路径

- `mcp/client.py` — `McpClientManager`：AsyncExitStack 管理、工具枚举、连接生命周期
- `mcp/config.py` — MCP 服务器配置解析（来自 `settings.json` 和 `plugin.json`）
- `tools/mcp_tool.py` — `McpToolAdapter`：将 MCP 工具包装为统一工具接口
- `skills/loader.py` — skill 文件加载、frontmatter 解析、多来源合并
- `skills/registry.py` — `SkillRegistry`：索引管理、system prompt 注入、按名查找
- `skills/types.py` — `SkillDefinition` 数据类型

## 设计亮点

- **统一插件格式**：`plugin.json` 将 skills、commands、hooks、MCP 配置聚合为单一 manifest，插件作者无需理解多套配置机制，分发和安装体验极简
- **Claude Code 生态兼容**：复用 `.claude-plugin/plugin.json` 规范，已有的 Claude Code 插件可直接在 OpenHarness 中使用，降低迁移成本
- **变量替换支持**：`${CLAUDE_PLUGIN_ROOT}` 使插件路径可移植，避免硬编码绝对路径的常见陷阱
- **Skills 无代码门槛**：纯 markdown 格式让非工程师也能编写和发布 skills，无需了解 Python 或 JSON schema

## 局限性

- **仅支持 stdio MCP 传输**：不支持 HTTP/SSE/WebSocket 等远程传输，所有 MCP 服务器必须作为本地子进程运行，无法连接远程或共享 MCP 服务
- **无 OAuth 支持**：缺乏 MCP 认证流程，连接需要认证的 MCP 服务器需要在配置中直接写入凭证
- **Skills 无动态参数**：frontmatter 不支持 `arguments` 字段声明，skill 调用时无法传入参数，每个 skill 只能执行固定逻辑
- **无 skill 热重载**：修改 `.md` 文件后需重启进程才能生效，与 hooks 的 mtime 热重载形成对比

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
