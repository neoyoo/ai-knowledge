---
title: "mcp-skills——agentscope"
category: L2
parent: "[[mcp-skills]]"
source: "agentscope"
source_version: "v2.0.1-11-g0e5418e8"
concept: "mcp-skills"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的 MCP 已统一为单个 `MCPClient` 类：通过 `StdioMCPConfig` / `HttpMCPConfig` 区分配置，通过 `is_stateful` 区分持久连接和每次调用临时连接。MCP 工具被包装成 `MCPTool`，并与普通 `ToolBase` 工具一起进入 `Toolkit` / `ToolGroup` 调度。

Skills 不是可直接调用的工具，而是 `Skill` / `SkillLoaderBase` 加载出的指令包。`Toolkit.get_skill_instructions()` 把可用 skill 的 name、description、dir 注入 prompt，并提供 `SkillViewer` 内置工具让 LLM 需要时读取完整 skill。

---

## 架构分析

### MCPClient

`MCPClient` 字段：

| 字段 | 作用 |
|---|---|
| `name` | MCP server 名称，也参与模型可见工具名 `mcp__{name}__{tool}` |
| `is_stateful` | 是否需要显式 `connect()` / `close()` 并复用 session |
| `mcp_config` | `StdioMCPConfig` 或 `HttpMCPConfig` |
| `enable_tools` / `disable_tools` | 工具白名单 / 黑名单 |
| `execution_timeout` | 单次 tool call timeout |

约束：

- stdio MCP 必须是 stateful
- HTTP MCP 可 stateful 或 stateless
- `enable_tools` 和 `disable_tools` 可同时存在，但不能有交集
- MCP client name 必须满足 provider 工具名字符约束

### 传输与会话

| 配置 | Stateful | Stateless |
|---|---|---|
| `StdioMCPConfig` | 支持；启动本地进程并持有 `ClientSession` | 不支持 |
| `HttpMCPConfig` | 支持；`connect()` 时创建 SSE 或 streamable HTTP session | 支持；每次 list/call 临时创建 session |

HTTP 传输自动根据 URL 选择：

- URL 以 `/sse` 或 `/messages/` 结尾：SSE client
- 其他 HTTP URL：streamable HTTP client

### MCPTool

`MCPClient.get_tool(name)` 返回 `MCPTool`：

- 模型可见工具名会被标准化为 `mcp__{mcp_name}__{sanitized_tool_name}`
- 原始 MCP tool name 保存在 `_tool.name`，实际调用 server 时仍用原名
- `inputSchema` 完整保留，包括 `$defs`、`anyOf`、`oneOf`
- `annotations.readOnlyHint` 会转为 `is_read_only`
- stateful 路径复用 `ClientSession`
- stateless 路径每次调用创建临时 session

### Toolkit / Skills 集成

`ToolGroup` 可以同时包含：

- `tools: list[ToolBase]`
- `skills_or_loaders: list[Skill | SkillLoaderBase]`
- `mcps: list[MCPClient]`

`Toolkit._get_available_tools()` 会在已激活 group 中收集 Python tools 与 MCP tools。`Toolkit._get_available_skills()` 则收集可用 skills，并由 `get_skill_instructions()` 渲染 `<agent-skills>` prompt。

---

## 关键代码路径

- `src/agentscope/mcp/_mcp_client.py` — unified `MCPClient`
- `src/agentscope/mcp/_config.py` — `StdioMCPConfig` / `HttpMCPConfig`
- `src/agentscope/tool/_adapters.py` — `MCPTool`
- `src/agentscope/tool/_toolkit.py` — `Toolkit._get_available_tools()` / `_get_available_skills()`
- `src/agentscope/tool/_tool_group.py` — `ToolGroup`
- `src/agentscope/skill/` — `Skill` 与 loader 抽象
- `src/agentscope/workspace/_local_workspace.py` — workspace 暴露 skills/MCPs

---

## 设计亮点

### 1. 单一 MCPClient 降低类层级复杂度

旧式 `StdIOStatefulClient` / `HttpStatefulClient` / `HttpStatelessClient` 类层级已经合并。现在通过 `mcp_config` + `is_stateful` 表达组合，调用方只需要处理一个 `MCPClient` 类型。

### 2. MCP 工具名做 provider-safe 标准化

LLM provider 通常限制工具名字符集。`MCPTool` 用 `mcp__{server}__{tool}` 命名，并把非法字符替换为 `x`，避免 MCP server 原始工具名导致 provider schema 拒绝。

### 3. 完整保留 MCP inputSchema

当前实现不再只复制 `properties` / `required`，而是保留完整 input schema，避免丢失 `$defs`、嵌套引用和 union 类型。

### 4. readOnlyHint 接入权限语义

MCP tool annotation 的 `readOnlyHint` 被转成 `is_read_only`，使 permission engine 可以对只读 MCP 工具自动允许或降低审批摩擦。

### 5. Skills 采用 prompt-index + viewer 延迟加载

系统 prompt 只注入 skill 摘要和目录，完整 skill 内容由 `SkillViewer` 工具按需读取，避免把所有 skill 全量塞入上下文。

---

## 局限性

### 1. Stateless HTTP 高频调用有连接开销

stateless 模式每次调用都新建 transport、`ClientSession`、`initialize()`。简单 API 封装可以接受，高频浏览器/状态型服务应使用 stateful。

### 2. 工具列表缓存无 TTL

`_cached_tools` 在 `list_raw_tools()` 后保留。MCP server 动态新增/删除工具时，需要显式刷新或重建 client，否则可能继续使用旧列表。

### 3. 只接入 MCP tools

当前页面核验到的是 tools 路径；MCP resources/prompts 没有进入 `Toolkit` 的同等一等抽象。

### 4. MCP 权限默认偏保守但不等于租户隔离

非 read-only MCP 默认 ASK，能防误操作，但 MCP server 进程权限、环境变量白名单、网络访问仍要在 workspace / deployment 层处理。

### 5. Skill 仍依赖模型遵循说明

Skill 是指令包，不是强类型工具。模型可能忘记读取、误读或不按 skill 执行；关键流程仍应升级为 Tool/MCP。

---

## 对 agent-os 的借鉴

agent-os 的扩展系统应分三层：

- `ToolProvider`: 本地工具、MCP tools、远程 agent tools 都产出统一 `ToolBase`
- `ExtensionBundle`: skills/prompts/resources 的描述性能力包，不直接等价为 tool
- `WorkspaceExtensionRegistry`: 每个 workspace/session 独立暴露 MCP 和 skills，避免全局工具池污染

MCP 侧建议采用 AgentScope 2.x 的“统一 client + stateful flag”方式，但补上 tool list TTL、resources/prompts 支持、环境变量白名单和租户级审批策略。

---

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
- 核心文件：
  - `src/agentscope/mcp/_mcp_client.py`
  - `src/agentscope/mcp/_config.py`
  - `src/agentscope/tool/_adapters.py`
  - `src/agentscope/tool/_toolkit.py`
  - `src/agentscope/tool/_tool_group.py`
  - `src/agentscope/skill/`
