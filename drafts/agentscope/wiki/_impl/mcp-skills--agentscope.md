---
title: "mcp-skills——agentscope"
category: L2
parent: "[[mcp-skills]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "mcp-skills"
created: "2026-04-15"
confidence: high
---

## 概述

AgentScope 把 MCP 实现为一套独立的客户端层（`agentscope/mcp/`），提供三种传输模式（StdIO/SSE/StreamableHTTP）× 两种会话策略（Stateful/Stateless）共五个具体客户端类。MCP 工具被包装为 `MCPToolFunction` 可调用对象，通过 `Toolkit.register_mcp_client()` 统一注册到工具集，与普通 Python 函数工具共享同一调度路径，支持 enable/disable 过滤、冲突处理策略和 postprocess hook。

## 架构分析

### 类层次

```
MCPClientBase (abc)
├── StatefulClientBase (abc)          # 长连接基类，AsyncExitStack 管理生命周期
│   ├── StdIOStatefulClient           # 本地进程，stdio 传输
│   └── HttpStatefulClient            # 远程 HTTP，SSE 或 StreamableHTTP 传输
└── HttpStatelessClient               # 无状态，每次调用新建/关闭 session
```

**核心抽象**：`MCPClientBase` 只定义两个责任：
1. `get_callable_function(func_name) -> MCPToolFunction` — 按名称获取工具
2. `_convert_mcp_content_to_as_blocks()` — MCP 内容块 → AgentScope 消息块（Text/Image/Audio/EmbeddedResource）

### 会话策略分化

| 策略 | 实现类 | 连接管理 | 适用场景 |
|------|--------|---------|---------|
| Stateful | `StatefulClientBase` 子类 | 显式 `connect()`/`close()`，单 session 跨多次调用 | 浏览器、有状态交互型 MCP server |
| Stateless | `HttpStatelessClient` | 每次工具调用新建 `ClientSession`，调用后关闭 | 无状态 HTTP API 型 MCP server |

### MCPToolFunction 双模式执行

`MCPToolFunction.__call__()` 根据构造时传入参数走两条路径：
- 传入 `client_gen`（无状态路径）：每次调用重新创建 `ClientSession` + `initialize()`
- 传入 `session`（有状态路径）：复用已有 session 直接 `session.call_tool()`

### 与 Toolkit 集成的数据流

```
MCP server
    ↓ list_tools() → List[mcp.types.Tool]
Toolkit.register_mcp_client()
    ↓ 逐个 get_callable_function() → MCPToolFunction
    ↓ register_tool_function(tool_func, group_name, preset_kwargs, ...)
Toolkit.tools: dict[str, RegisteredToolFunction]   # 与普通 Python 工具共存
    ↓ _execute_tool_in_background()
MCPToolFunction.__call__(**kwargs)
    ↓ session.call_tool() → mcp.types.CallToolResult
    ↓ _convert_mcp_content_to_as_blocks()
ToolResponse(content=[TextBlock | ImageBlock | AudioBlock], metadata=res.meta)
```

### JSON Schema 提取

`_extract_json_schema_from_mcp_tool(tool)` 将 `mcp.types.Tool.inputSchema` 转为 OpenAI function-calling 格式：

```python
{
    "type": "function",
    "function": {
        "name": tool.name,
        "description": tool.description,
        "parameters": {
            "type": "object",
            "properties": tool.inputSchema.get("properties", {}),
            "required": tool.inputSchema.get("required", []),
        },
    },
}
```

这份 schema 直接写入 `MCPToolFunction.json_schema`，供 LLM 工具调用时使用。

## 关键代码路径

### 有状态客户端连接建立

```python
# _stateful_client_base.py: StatefulClientBase.connect()
async def connect(self) -> None:
    self.stack = AsyncExitStack()
    context = await self.stack.enter_async_context(self.client)
    read_stream, write_stream = context[0], context[1]
    self.session = ClientSession(read_stream, write_stream)
    await self.stack.enter_async_context(self.session)
    await self.session.initialize()
    self.is_connected = True
```

- `self.client` 在子类构造函数中设置（StdIO 用 `stdio_client(StdioServerParameters(...))`，HTTP 用 `streamablehttp_client(...)` 或 `sse_client(...)`）
- `AsyncExitStack` 确保两层上下文（transport + session）按 LIFO 顺序关闭

### 工具调用执行路径（无状态）

```python
# _mcp_function.py: MCPToolFunction.__call__()
async def __call__(self, **kwargs) -> mcp.types.CallToolResult | ToolResponse:
    if self.client_gen:                                          # 无状态路径
        async with self.client_gen() as cli:
            read_stream, write_stream = cli[0], cli[1]
            async with ClientSession(read_stream, write_stream) as session:
                await session.initialize()
                res = await session.call_tool(
                    self.name, arguments=kwargs,
                    read_timeout_seconds=self.timeout,
                )
    else:                                                        # 有状态路径
        res = await self.session.call_tool(
            self.name, arguments=kwargs,
            read_timeout_seconds=self.timeout,
        )

    if self.wrap_tool_result:
        as_content = MCPClientBase._convert_mcp_content_to_as_blocks(res.content)
        return ToolResponse(content=as_content, metadata=res.meta)
    return res
```

### 工具注册到 Toolkit

```python
# tool/_toolkit.py: Toolkit.register_mcp_client()
async def register_mcp_client(
    self,
    mcp_client: MCPClientBase,
    group_name: str = "basic",
    enable_funcs: list[str] | None = None,
    disable_funcs: list[str] | None = None,
    preset_kwargs_mapping: dict[str, dict] | None = None,
    postprocess_func: Callable | None = None,
    namesake_strategy: Literal["override", "skip", "raise", "rename"] = "raise",
) -> None:
    for mcp_tool in await mcp_client.list_tools():
        if enable_funcs and mcp_tool.name not in enable_funcs: continue
        if disable_funcs and mcp_tool.name in disable_funcs:  continue

        func_obj = await mcp_client.get_callable_function(
            func_name=mcp_tool.name, wrap_tool_result=True,
        )
        preset_kwargs = preset_kwargs_mapping.get(mcp_tool.name, {}) if preset_kwargs_mapping else None
        self.register_tool_function(
            tool_func=func_obj, group_name=group_name,
            preset_kwargs=preset_kwargs, postprocess_func=postprocess_func,
            namesake_strategy=namesake_strategy,
        )
```

### MCP 工具移除

```python
# tool/_toolkit.py: Toolkit.remove_mcp_clients()
async def remove_mcp_clients(self, client_names: list[str]) -> None:
    for func_name in list(self.tools.keys()):
        if self.tools[func_name].mcp_name in client_names:
            self.tools.pop(func_name)
```

`RegisteredToolFunction` 保存了 `mcp_name` 字段，使得可以按 server 维度批量移除工具。

### 错误处理

`Toolkit._execute_tool_in_background()` 捕获 `mcp.shared.exceptions.McpError`，将其包装为 `ToolResponse(content=[TextBlock(text=f"Error: {e}")])` 而非抛出异常，保证 agent loop 不中断。

## 设计亮点

### 1. 传输层与会话策略正交分离

三种传输（StdIO/SSE/StreamableHTTP）和两种会话策略（Stateful/Stateless）被清晰分离：
- `StdIOStatefulClient` 只管设置 `self.client = stdio_client(...)`，连接逻辑全在基类
- `HttpStatefulClient` 和 `HttpStatelessClient` 共享相同的 HTTP 传输配置参数，但生命周期完全不同

这使得增加新传输类型只需实现一个最小子类。

### 2. MCPToolFunction 统一封装

MCP 工具被包装为普通的 `async def __call__(**kwargs)` 对象，与 Python 函数工具共享 Toolkit 注册路径。LLM 不感知底层是 MCP 还是本地函数，JSON Schema 格式完全统一。

### 3. 工具粒度控制

`register_mcp_client()` 提供四个维度的精细控制：
- `enable_funcs`/`disable_funcs`：白名单/黑名单过滤
- `preset_kwargs_mapping`：为特定工具预置参数（如全局 auth token）
- `postprocess_func`：工具结果后处理（可 sync 或 async）
- `namesake_strategy`：同名冲突处理（raise/override/skip/rename）

这是多数 MCP 集成方案中罕见的精细度。

### 4. 多模态内容自动转换

`_convert_mcp_content_to_as_blocks()` 处理 MCP 返回的 Text/Image/Audio/EmbeddedResource 四种内容类型，自动转为 AgentScope 消息块体系，支持多模态 agent 直接消费 MCP 工具结果。

### 5. 按 MCP server 批量移除

`remove_mcp_clients(client_names)` 通过 `RegisteredToolFunction.mcp_name` 字段实现按 server 维度批量卸载，支持动态工具集管理（如断开某个 MCP server 后清理其所有工具）。

## 局限性

### 1. 无状态客户端每次调用重建 session 开销高

`HttpStatelessClient` 每次 `__call__` 都完整走 `get_client() → ClientSession → initialize() → call_tool` 流程，高频调用时延迟显著，无连接池或复用机制。

### 2. 多个有状态客户端关闭必须严格 LIFO

代码注释明确指出，多个 `StdIOStatefulClient` 或 `HttpStatefulClient` 并存时，必须按 Last In First Out 顺序关闭，否则可能出错（上游 MCP Python SDK 已知问题 #577）。这要求使用方手动维护关闭顺序，容易出错。

### 3. 工具列表缓存无失效机制

`StatefulClientBase._cached_tools` 在首次 `list_tools()` 后永久缓存，MCP server 动态添加/删除工具后不会自动刷新。Stateless 客户端 `HttpStatelessClient._tools` 同样无 TTL。

### 4. EmbeddedResource BlobResourceContents 未支持

`_convert_mcp_content_to_as_blocks()` 对 `mcp.types.BlobResourceContents` 直接 `logger.error` 跳过，代码中有明确 TODO，二进制资源（如 PDF、任意文件）无法通过 MCP 工具结果传递。

### 5. `enable_funcs` 和 `disable_funcs` 不能同时指定

注册时若同时传入 `enable_funcs` 和 `disable_funcs`，代码做了 assert 检查，但实际上 enable 优先级语义更自然，二者互斥是不必要的限制。

### 6. 无资源（Resources）和提示（Prompts）支持

AgentScope MCP 客户端目前只实现了 `tools`，MCP 协议中的 `resources`（结构化数据资源）和 `prompts`（可复用提示模板）两个能力完全未接入。

## 来源

- 源码版本：`0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12`
- 分析深度：源码级
- 主要文件：
  - `src/agentscope/mcp/__init__.py`
  - `src/agentscope/mcp/_client_base.py`
  - `src/agentscope/mcp/_mcp_function.py`
  - `src/agentscope/mcp/_stateful_client_base.py`
  - `src/agentscope/mcp/_stdio_stateful_client.py`
  - `src/agentscope/mcp/_http_stateless_client.py`
  - `src/agentscope/mcp/_http_stateful_client.py`
  - `src/agentscope/tool/_toolkit.py`（`register_mcp_client`、`remove_mcp_clients`）
  - `src/agentscope/_utils/_common.py`（`_extract_json_schema_from_mcp_tool`）
  - `examples/functionality/mcp/main.py`（使用示例）
