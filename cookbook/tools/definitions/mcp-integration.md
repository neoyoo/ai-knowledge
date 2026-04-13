---
tool-category: mcp-integration
tags: [mcp, model-context-protocol, resources, tool-discovery, extensibility]
sources: [claude-code, deer-flow, openharness]
---

# MCP Integration Tools

> 通过 Model Context Protocol 发现、加载、调用外部工具的工具集——解决"工具太多放不进 context"的工程问题。

## 本质

MCP（Model Context Protocol）解决了一个矛盾：外部工具数量可能非常多（几十到几百个），但把全部 schema 塞进 context 会撑爆 token 预算，模型也难以选择。

这类工具的设计目标是：**按需发现**——context 里只放工具索引，需要哪个再加载哪个的完整 schema，实现"延迟加载"。三个项目对这个问题有截然不同的解法。

## 工具总览

| 工具名 | 来源 | 核心作用 |
|--------|------|----------|
| `ListMcpResources` / `list_mcp_resources` | Claude Code / DeerFlow | 列出可用 MCP 资源 |
| `ReadMcpResource` / `read_mcp_resource` | Claude Code / DeerFlow | 读取特定 MCP 资源内容 |
| `ToolSearch` / `tool_search` | Claude Code / DeerFlow | 查找并加载 deferred tools 的 schema |
| `mcp_auth` | OpenHarness | 配置 MCP server 认证凭证 |
| `mcp__{server}__{tool}` | OpenHarness（动态生成） | McpToolAdapter 自动创建的 MCP 工具 |

---

## 工具 Schema（Anthropic tool_use 格式）

### ListMcpResources / list_mcp_resources — 列出 MCP 资源

Claude Code 和 DeerFlow 都将 MCP Resources 作为一等公民，与 MCP Tools 并列。资源是只读数据源（文件、数据库记录、API 响应缓存等）。

```json
{
  "name": "list_mcp_resources",
  "description": "List all available MCP resources from connected servers. Resources are read-only data sources (files, database records, cached API responses) distinct from executable tools.",
  "input_schema": {
    "type": "object",
    "properties": {
      "server_name": {
        "type": "string",
        "description": "Filter resources from a specific MCP server. Omit to list resources from all connected servers."
      }
    },
    "required": []
  }
}
```

---

### ReadMcpResource / read_mcp_resource — 读取 MCP 资源

```json
{
  "name": "read_mcp_resource",
  "description": "Read the content of a specific MCP resource by URI. Returns the resource content (text, binary, or structured data) depending on the resource type.",
  "input_schema": {
    "type": "object",
    "properties": {
      "uri": {
        "type": "string",
        "description": "The resource URI returned by list_mcp_resources. Format varies by server (e.g. 'file:///path/to/doc', 'db://table/id', 'https://api.example.com/resource/123')."
      },
      "server_name": {
        "type": "string",
        "description": "The MCP server to read from. Required when the URI is ambiguous across multiple servers."
      }
    },
    "required": ["uri"]
  }
}
```

---

### ToolSearch / tool_search — 查找并加载延迟工具

这是 MCP Integration 工具集里最关键的工具，三个项目的实现差异最大。

**Claude Code 版本**（支持精确选择 + 关键词搜索）：

```json
{
  "name": "ToolSearch",
  "description": "Find and load the full schema of deferred tools. Deferred tools are registered by name only until queried — their full parameter schema is not loaded into context until you call this tool. Use this before calling any tool marked as deferred.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search query to find deferred tools. Two formats supported:\n- 'select:ToolName1,ToolName2' — exact lookup by name (fastest, use when you know the tool name)\n- 'keyword1 keyword2' — semantic keyword search, returns best matches ranked by relevance\n\nExamples:\n- 'select:Read,Edit,Grep' — load schemas for these exact tools\n- 'jupyter notebook execute' — find notebook execution tools\n- '+slack send message' — require 'slack' in name, rank by remaining terms"
      },
      "max_results": {
        "type": "integer",
        "description": "Maximum number of tool schemas to return. Default 5. Increase for broad keyword searches.",
        "default": 5
      }
    },
    "required": ["query"]
  }
}
```

**DeerFlow 版本**（支持 select:/ + keyword / free-text regex）：

```json
{
  "name": "tool_search",
  "description": "Search the deferred tool registry to find and load tool schemas. When tool_search.enabled=true, MCP tools are not injected into the system prompt — use this tool to discover and load them on demand.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Search query. Supports three formats:\n- 'select:/tool_name' — exact match by tool name\n- '+required_term optional_terms' — require term in name, rank by others\n- 'free text regex' — full-text search across tool names and descriptions\n\nExamples:\n- 'select:/web_search' — load web_search schema\n- '+github create issue' — find GitHub issue creation tools\n- 'database query sql' — find any SQL or database tools"
      }
    },
    "required": ["query"]
  }
}
```

**OpenHarness 版本**（简单 substring 匹配，MCP 工具已全量注入）：

```json
{
  "name": "tool_search",
  "description": "Search available tools by name substring. Unlike Claude Code and DeerFlow, OpenHarness injects all MCP tool schemas upfront — this tool is used to discover and recall tool names, not to lazily load schemas.",
  "input_schema": {
    "type": "object",
    "properties": {
      "query": {
        "type": "string",
        "description": "Substring to match against tool names. Case-insensitive. Returns all tools whose name contains this string."
      }
    },
    "required": ["query"]
  }
}
```

---

### mcp_auth — MCP 认证配置（OpenHarness 独有）

OpenHarness 提供显式的认证配置工具，允许在运行时为特定 MCP server 配置凭证，无需重启。

```json
{
  "name": "mcp_auth",
  "description": "[OpenHarness only] Configure authentication credentials for an MCP server at runtime. Supports bearer token, custom header, and environment variable injection.",
  "input_schema": {
    "type": "object",
    "properties": {
      "server_name": {
        "type": "string",
        "description": "The MCP server to configure authentication for. Must match a server name in settings.json."
      },
      "auth_type": {
        "type": "string",
        "description": "Authentication mechanism to use.",
        "enum": ["bearer", "header", "env"]
      },
      "token": {
        "type": "string",
        "description": "Bearer token value. Required when auth_type='bearer'."
      },
      "header_name": {
        "type": "string",
        "description": "Custom header name. Required when auth_type='header'."
      },
      "header_value": {
        "type": "string",
        "description": "Custom header value. Required when auth_type='header'."
      },
      "env_var": {
        "type": "string",
        "description": "Environment variable name containing the credential. Required when auth_type='env'. The variable is read from the process environment at call time."
      }
    },
    "required": ["server_name", "auth_type"]
  }
}
```

---

### mcp__{server}__{tool} — 动态生成的 MCP 工具（OpenHarness）

OpenHarness 的 `McpToolAdapter` 为每个 MCP server 的每个工具自动生成一个同名工具，命名规则：`mcp__<server_name>__<tool_name>`。

这类工具的 schema 由 MCP server 的 `list_tools()` 响应动态生成，不需要手写。以下是一个典型例子（假设连接了名为 `filesystem` 的 MCP server，提供 `read_file` 工具）：

```json
{
  "name": "mcp__filesystem__read_file",
  "description": "[Auto-generated by McpToolAdapter] Read a file from the filesystem MCP server.",
  "input_schema": {
    "type": "object",
    "properties": {
      "path": {
        "type": "string",
        "description": "Absolute or relative path to the file to read."
      }
    },
    "required": ["path"]
  }
}
```

命名规则让模型可以通过工具名直接识别来源 server，无需额外文档。

---

## 跨项目对比

### 延迟加载策略

这是三个项目在 MCP 集成上差异最大的设计决策。

| 维度 | Claude Code | DeerFlow | OpenHarness |
|------|-------------|----------|-------------|
| **策略** | 延迟加载（shouldDefer 判断） | 延迟加载（tool_search.enabled 开关） | 全量注入 |
| **默认行为** | 部分工具标记为 deferred | 可配置，建议 MCP 工具延迟 | 所有 MCP 工具全量注入 system prompt |
| **发现方式** | ToolSearch（支持精确 select + 关键词） | tool_search（支持 regex + keyword） | 全已可见，用 tool_search 做召回 |
| **schema 加载时机** | 调用 ToolSearch 后才加载进 context | 调用 tool_search 后才加载进 context | 启动时全量加载 |
| **适用场景** | MCP 工具 10-100 个时 token 预算受控 | 大规模 MCP 集成（企业内部工具集） | MCP 工具少（< 20 个），优先简单性 |
| **额外调用轮次** | +1 轮（先 ToolSearch 再调工具） | +1 轮（先 tool_search 再调工具） | 无额外轮次 |

### 命名规范

| 项目 | 本地工具 | MCP 工具 | 区分方式 |
|------|----------|----------|----------|
| Claude Code | `Read`, `Bash`, `Agent` | 与本地工具统一接口，MCPTool 包装 | 通过工具类型标记，非命名区分 |
| DeerFlow | `web_search`, `task` | 保留原始工具名 | 无显式前缀，通过 server config 区分 |
| OpenHarness | `read_file`, `bash` | `mcp__servername__toolname` | 前缀 `mcp__` 明确标记来源 |

### 认证机制

| 项目 | 支持的认证方式 | 配置方式 | 运行时更新 |
|------|----------------|----------|------------|
| Claude Code | OAuth（Bearer + refresh token）| 自动处理 token 刷新和 401 重试 | 支持（OAuth 流程自动续期） |
| DeerFlow | OAuth 内置 | `extensions_config.json` 中配置 | 支持 |
| OpenHarness | Bearer / Header / Env var | `mcp_auth` 工具运行时配置 | 支持（工具调用即更新） |

### 资源 vs 工具

Claude Code 明确区分 MCP Resources（只读数据）和 MCP Tools（可执行操作），提供专用的 `ListMcpResources` / `ReadMcpResource`。DeerFlow 沿用此模型。OpenHarness 通过 `McpClientManager` 的 `list_resources()` 枚举资源，但没有专用工具暴露给模型，需要通过对应的 MCP 工具间接访问。

---

## 最佳实践

### 1. 先 ToolSearch，再调工具

永远不要猜测 deferred 工具的参数 schema。先用 ToolSearch 加载 schema，确认参数名和类型，再调用。

```text
# 正确顺序
1. ToolSearch: query="select:GitCommit,GitDiff"
   → 拿到完整 schema，确认参数
2. GitCommit: message="fix: ...", files=[...]

# 错误做法（凭感觉调，参数名可能完全不对）
GitCommit: commit_message="fix: ..."  # 猜错参数名导致调用失败
```

### 2. 用 select: 前缀精确加载，避免关键词歧义

当你已经知道工具名时，用 `select:` 直接指定，比关键词搜索更快、更精准。

```text
# 精确：已知工具名
ToolSearch: query="select:Read,Edit,Grep"

# 关键词搜索：不确定工具名时使用
ToolSearch: query="notebook jupyter execute cell"

# +required_term：要求名字包含某词
ToolSearch: query="+github list repositories"
```

### 3. 控制 MCP 工具数量，优先延迟加载

无论哪个项目，当 MCP tools 超过 20 个时，建议开启延迟加载（Claude Code `shouldDefer`，DeerFlow `tool_search.enabled=true`）。全量注入会消耗大量 context，模型也难以从 100 个工具里选择正确的一个。

经验阈值：
- MCP 工具 ≤ 10 个：全量注入可接受（OpenHarness 模式）
- MCP 工具 11-50 个：建议延迟加载
- MCP 工具 > 50 个：必须延迟加载，否则 context 溢出

### 4. OpenHarness 的 mcp__ 命名前缀要带进 prompt

使用 OpenHarness 时，系统 prompt 或任务描述里提到工具要用全名（含 `mcp__` 前缀），否则模型容易混淆同名的本地工具和 MCP 工具。

```text
# 明确
"使用 mcp__filesystem__read_file 读取配置文件"

# 模糊（read_file 可能是本地工具也可能是 MCP 工具）
"使用 read_file 读取配置文件"
```

### 5. 用 ReadMcpResource 替代 MCP 工具读取静态资源

如果 MCP server 提供了资源（Resources），优先用 `ReadMcpResource` 而非通过工具间接获取。Resources 是只读的，不消耗工具调用的 turn，且语义更清晰。

```python
# 偏好：用 Resource 读取静态文档
list_mcp_resources()  # 发现可用资源
read_mcp_resource(uri="docs://api/reference")  # 直接读取

# 次选：通过工具读取（工具调用有 overhead）
mcp__docs_server__fetch_api_reference()
```

### 6. 认证凭证用环境变量，不要硬编码

使用 OpenHarness 的 `mcp_auth` 时，选 `env` 模式而非直接传 token，避免凭证出现在 transcript 中。

```python
# 安全：从环境变量读取
mcp_auth(server_name="github", auth_type="env", env_var="GITHUB_TOKEN")

# 不安全：token 明文出现在工具调用记录里
mcp_auth(server_name="github", auth_type="bearer", token="ghp_xxxxxxxxxxxx")
```

---

## 常见踩坑

**1. Deferred 工具 schema 未加载就调用**

调用 deferred 工具时如果没有先用 ToolSearch 加载 schema，调用会失败，报 `InputValidationError` 或参数校验错误。

解决：养成习惯——任何 deferred 工具都先 ToolSearch，拿到完整 schema 后再调用。

**2. DeerFlow tool_search.enabled=false 时延迟加载不生效**

如果 `extensions_config.json` 里没有显式设置 `tool_search.enabled: true`，DeerFlow 会全量注入所有 MCP 工具 schema，不走延迟加载路径。工具数量多时会导致 context 溢出。

解决：有超过 10 个 MCP 工具时，显式开启 `tool_search.enabled: true`。

**3. MCP server 崩溃后 Claude Code 需要手动重连**

Claude Code 的 MCP client 目前没有自动重连机制。server 崩溃后，工具调用会静默失败或返回连接错误，需要手动执行 `/reload-plugins` 重新初始化 MCP 连接。

解决：在生产使用中监控 MCP server 进程健康状态；对关键 MCP server 用 supervisor 等工具保证进程存活。

**4. ReadMcpResource 大输出被截断**

Claude Code 的 `mcpContentNeedsTruncation` 会截断超大的 MCP 资源内容，向 agent 返回持久化路径而非原始内容。agent 需要通过路径再次读取完整内容。

解决：读取大型资源后，检查返回结果是否包含文件路径引用，有则用 Read 工具跟进读取。

**5. OpenHarness 全量注入与工具过多的 context 压力**

OpenHarness 默认把所有 MCP 工具 schema 注入 system prompt，工具多时会显著挤压可用 context。

解决：控制连接的 MCP server 数量；或在 `settings.json` 中为不常用的 server 设置 `enabled: false`，按需开启。

---

## 关联

关联 wiki：[[wiki/tool-system]] · [[wiki/mcp-skills]] · [[wiki/context-management]]

关联 cookbook：[[cookbook/tools/definitions/skill-system]] · [[cookbook/tools/definitions/agent-orchestration]]
