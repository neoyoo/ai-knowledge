---
title: "MCP & Skills — DeerFlow"
category: L2
parent: "[[mcp-skills]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 在 MCP 和 Skills 两个维度都有独特设计。MCP 层面，通过 `langchain-mcp-adapters` 支持多服务器连接，并引入"延迟工具注册"机制解决大规模 MCP 工具的 token 膨胀问题。Skills 层面，用带 YAML frontmatter 的 Markdown 文件定义可复用的任务流程，并支持 agent 在完成复杂任务后自动创建新 skill，实现自我进化。

## 架构分析

### MCP 客户端架构

DeerFlow 使用 `langchain-mcp-adapters` 的 `MultiServerMCPClient`，在单个客户端实例中管理多个 MCP 服务器连接。支持三种 transport：

- `stdio`：本地进程，适合本地工具服务
- `sse`：Server-Sent Events，适合长连接流式场景
- `http`：标准 HTTP，适合远程 MCP 服务

配置集中在 `extensions_config.json` 的 `mcpServers` 键下。MCP 工具列表使用 **mtime-based 缓存**，文件未变更时跳过重新加载，降低启动开销。内置 OAuth 支持，可对接需要授权的 MCP 服务器。

### 延迟工具注册（Deferred Tool Registry）

当 `tool_search.enabled=True` 时，MCP 工具不会全量注入 system prompt。取而代之的是：

1. System prompt 只包含工具索引（名称 + 简短描述）
2. Agent 通过 `tool_search` 工具按需查找工具
3. 找到目标工具后，完整 schema 才加载进上下文

这是解决"MCP 工具爆炸导致 context 溢出"的工程方案，在所有分析过的项目中是唯一采用此策略的实现。

### Skills 文件格式

每个 skill 是一个 Markdown 文件，YAML frontmatter 定义元数据：

```yaml
---
name: deep-research
description: "Conducts multi-step research with web search and synthesis"
allowed-tools:
  - web_search
  - read_url
  - write_file
---
```

正文是自然语言的任务流程描述，agent 按需读取后作为任务指导。

### 两级 Skills 目录

- `skills/public/`：20 个内置 skill，覆盖场景包括 `deep-research`、`ppt-generation`、`frontend-design`、`data-analysis` 等
- `skills/custom/`：用户自定义 skill，不受版本更新影响

### 懒加载策略

Agent 的 system prompt 中只包含所有 skill 的索引（名称 + description），不加载全文。当任务需要某个 skill 时，通过 `read_file` 工具按需读取完整内容。这与 MCP 的延迟工具注册思路一致——都是为了控制 context 占用。

### Skill 自进化机制

启用 `skill_evolution` 后，agent 在完成复杂任务后会调用 `skill_manage` 工具：

- 如果该任务流程可复用 → 创建新 skill 文件写入 `skills/custom/`
- 如果已有 skill 需改进 → 更新对应文件

自进化的质量完全依赖 LLM 自我反思能力，没有外部验证机制。

### 关键代码路径

- `deerflow/mcp/` — MCP 客户端、OAuth 支持、缓存逻辑
- `deerflow/skills/loader.py` — Skills 索引构建与懒加载
- `deerflow/tools/builtins/skill_manage_tool.py` — Skill 创建/更新工具实现

## 设计亮点

- **延迟工具注册唯一性**：在分析过的所有项目中，DeerFlow 是唯一将 MCP 工具从 context 中剥离、按需检索的实现，从根本上解决了大规模 MCP 集成时的 token 膨胀问题
- **Skill 自进化**：agent 通过 `skill_manage` 工具积累可复用的任务流程，知识库随使用增长，形成正向飞轮
- **20 个内置 skill 覆盖多场景**：开箱即用的高价值场景（PPT 生成、前端设计、数据分析），降低用户使用门槛
- **OAuth 内置**：企业级 MCP 服务的授权需求开箱支持，不需要额外封装

## 局限性

- **Skills 无结构化参数**：skill 只是自然语言文本，没有类似函数签名的参数定义，调用时无法做参数校验或自动补全
- **Skill 自进化质量不可控**：LLM 自动创建的 skill 可能质量参差不齐，没有评审或测试机制保证可靠性
- **延迟工具注册增加调用轮次**：agent 需要先调 `tool_search` 再调实际工具，对简单任务引入不必要的 RTT
- **mtime 缓存粒度较粗**：配置文件局部更新时仍需全量重载 MCP 工具列表

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
