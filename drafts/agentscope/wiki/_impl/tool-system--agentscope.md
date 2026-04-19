---
title: "tool-system——agentscope"
category: L2
parent: "[[tool-system]]"
source: agentscope
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: tool-system
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

AgentScope 的工具系统以 `Toolkit` 类为核心，统一管理三类能力来源：普通 Python 函数、MCP 服务端工具、Agent Skill 目录。工具以「工具组（ToolGroup）」为粒度做动态启用/停用，执行层面全部统一为 `AsyncGenerator[ToolResponse, None]` 流式接口，并叠加洋葱式 middleware 和 OpenTelemetry 追踪。这套设计的独特价值在于：工具的 JSON schema 自动从 docstring 提取、工具组状态可被 LLM 自己通过元工具 `reset_equipped_tools` 在运行时切换、MCP 同时支持有状态/无状态两种客户端模式。

---

## 架构分析

### 整体分层

```
Toolkit (StateModule)
  ├── tools: dict[str, RegisteredToolFunction]   # 所有已注册工具
  ├── groups: dict[str, ToolGroup]               # 工具分组元数据
  ├── skills: dict[str, AgentSkill]              # Agent Skill 目录索引
  └── _middlewares: list                         # 洋葱中间件链

RegisteredToolFunction (dataclass)
  ├── name / group / source                       # 元信息
  ├── original_func: ToolFunction                 # 原始可调用对象
  ├── json_schema: dict                           # LLM 看到的 schema
  ├── preset_kwargs: dict                         # 预设参数（不暴露给 LLM）
  ├── extended_model: Type[BaseModel] | None      # 动态扩展 schema 用
  ├── postprocess_func: Callable | None           # 执行后后处理钩子
  └── async_execution: bool                       # 是否后台异步执行

ToolResponse (dataclass)
  ├── content: List[TextBlock | ImageBlock | AudioBlock | VideoBlock]
  ├── metadata: dict | None
  ├── stream: bool / is_last: bool / is_interrupted: bool
  └── id: str
```

`Toolkit` 继承自 `StateModule`，可以通过 `state_dict()` / `load_state_dict()` 序列化/恢复工具组激活状态，支持跨 session 持久化。

### 工具来源三路径

**路径 1：Python 函数**

`register_tool_function(tool_func, ...)` 接受普通函数、`partial` 包装函数或 `MCPToolFunction` 对象。对于普通/partial 函数，调用 `_parse_tool_function()` 从 docstring 和函数签名自动提取 JSON schema：用 `docstring_parser.parse()` 解析 Google/Numpy 风格的 docstring，用 `inspect.signature()` 遍历参数，通过 Pydantic `create_model()` 动态构造参数模型，最终输出 `{"type": "function", "function": {"name": ..., "description": ..., "parameters": {...}}}` 格式的 schema。

**路径 2：MCP 客户端**

`register_mcp_client(mcp_client, ...)` 调用 `mcp_client.list_tools()` 获取工具列表，对每个工具调用 `mcp_client.get_callable_function()` 获得 `MCPToolFunction` 实例，再统一走 `register_tool_function()`。MCP 工具的 schema 由 `_extract_json_schema_from_mcp_tool()` 从 `mcp.types.Tool.inputSchema` 提取，不需要 docstring。

MCP 客户端有两种模式：
- `HttpStatelessClient`（SSE / streamable HTTP）：每次工具调用都新建 session，完全无状态
- `StdIOStatefulClient` / `HttpStatefulClient`：通过 `AsyncExitStack` 维持长连接 session，支持 `connect()` / `close()` 生命周期管理，适合需要保持状态的 MCP 服务（如浏览器控制）

**路径 3：Agent Skill**

`register_agent_skill(skill_dir)` 扫描目录下的 `SKILL.md`，用 `python-frontmatter` 读取 YAML front matter 中的 `name` 和 `description` 字段，存入 `self.skills`。不产生可调用工具，而是通过 `get_agent_skill_prompt()` 生成 prompt 片段注入 system prompt，引导 LLM 自行 `Read` skill 目录下的文件。

### 工具组（ToolGroup）动态管理

工具分为 `"basic"` 组（永远激活）和自定义命名组。`get_json_schemas()` 只返回 `group == "basic"` 或 `groups[group].active == True` 的工具 schema。

元工具 `reset_equipped_tools(**kwargs: bool)` 是一个内置的 self-modifying 工具，LLM 可以在运行时通过调用它来改变自身可见的工具集。`get_json_schemas()` 在返回 schema 前会给 `reset_equipped_tools` 动态注入所有非 basic 组的字段（通过 `extended_model` 机制），使其参数与当前分组配置保持同步。

### 执行流水线

`call_tool_function(tool_call: ToolUseBlock)` 的装饰器叠加顺序（从外到内）：

```
@trace_toolkit            # OpenTelemetry span，最外层保证可观测性完整
@_apply_middlewares       # 洋葱中间件链
async def call_tool_function(self, tool_call)
```

执行内部逻辑：
1. 查找工具，检查组是否激活
2. 合并 `preset_kwargs` 与 `tool_call["input"]`
3. 若 `async_execution=True`：`asyncio.create_task()` 后台执行，立即返回 task_id
4. 否则同步执行：区分 coroutine / async generator / sync function / sync generator，统一包装成 `AsyncGenerator[ToolResponse, None]`
5. postprocess_func 在每个 chunk 上应用

`_async_wrapper.py` 中三个包装函数统一返回类型：
- `_object_wrapper`：`ToolResponse → AsyncGenerator`
- `_sync_generator_wrapper`：`Generator[ToolResponse] → AsyncGenerator`
- `_async_generator_wrapper`：`AsyncGenerator[ToolResponse] → AsyncGenerator`（含 CancelledError 转化为 `is_interrupted=True`）

## 关键代码路径

- `src/agentscope/tool/_toolkit.py` — `Toolkit` 主类，注册/管理/执行全在此
- `src/agentscope/tool/_types.py` — `RegisteredToolFunction`、`ToolGroup`、`AgentSkill` 数据结构
- `src/agentscope/tool/_response.py` — `ToolResponse` 统一响应格式
- `src/agentscope/tool/_async_wrapper.py` — 三路返回类型统一为 AsyncGenerator
- `src/agentscope/mcp/_mcp_function.py` — `MCPToolFunction` 包装 MCP 工具为可调用对象
- `src/agentscope/mcp/_client_base.py` — `MCPClientBase`，MCP 内容到 AS blocks 的转换
- `src/agentscope/mcp/_stateful_client_base.py` — 有状态客户端基类，`connect()/close()` 生命周期
- `src/agentscope/mcp/_http_stateless_client.py` — 无状态 HTTP 客户端
- `src/agentscope/mcp/_stdio_stateful_client.py` — stdio 有状态客户端
- `src/agentscope/tracing/_trace.py` — `trace_toolkit` 装饰器，OpenTelemetry span
- `src/agentscope/_utils/_common.py:_parse_tool_function` — docstring → JSON schema 提取

---

## 设计亮点

**1. 工具组激活状态由 LLM 自己管理（元工具范式）**

`reset_equipped_tools` 是一个被注册为普通工具的 Python 方法。它的参数 schema 由 `get_json_schemas()` 每次调用时动态注入（通过 Pydantic `create_model` 动态构造 extended_model），保证 LLM 看到的参数与当前工具组配置完全同步。LLM 可以在运行时自主决定激活哪些工具组，而不需要重新初始化 Toolkit，这是一种将工具管理权「下放给 LLM」的设计。

**2. preset_kwargs 实现工具参数隐藏**

注册时通过 `preset_kwargs` 预设的参数会从 JSON schema 中剔除，LLM 无法看到也无法修改这些参数（如 API key、session_id 等）。执行时再由 Toolkit 自动合并：`kwargs = {**tool_func.preset_kwargs, **tool_call["input"]}`。这种「部分暴露」模式避免了为每种工具包装 wrapper 函数。

**3. extended_model 动态扩展 schema**

`set_extended_model(func_name, model: Type[BaseModel])` 可以在运行时给已注册工具附加额外参数，Pydantic `$defs` 冲突检测逻辑确保合并安全。这为「工具参数由上下文决定」的场景（如 `reset_equipped_tools` 的组参数）提供了通用机制，无需修改原函数签名。

**4. MCP 有状态/无状态双模式**

两种 MCP 客户端模式在接口上完全一致（都暴露 `list_tools()` / `get_callable_function()`），但内部实现截然不同：无状态客户端每次工具调用新建 session，无连接泄漏风险；有状态客户端通过 `AsyncExitStack` 维持长连接，适合 browser-use 等需要跨调用保持上下文的场景。两者对 Toolkit 透明，注册路径完全相同。

**5. 流式执行统一接口**

所有工具执行结果（单对象、同步生成器、异步生成器）都被统一包装为 `AsyncGenerator[ToolResponse, None]`。这让 Toolkit 的调用方（Agent 的 reply 循环）无需区分工具是否流式，统一 `async for chunk in await toolkit.call_tool_function(...)` 消费即可。

**6. 后台异步执行（实验性）**

`async_execution=True` 的工具会被 `asyncio.create_task()` 投入后台运行，立即返回 task_id。LLM 可通过 `view_task/wait_task/cancel_task` 查询状态。这为长时工具（如代码执行、文件下载）提供了非阻塞执行路径，避免占用 Agent 的主 await 链。

**7. Agent Skill prompt 注入范式**

Agent Skill 不产生工具调用，而是生成 prompt 片段。`get_agent_skill_prompt()` 返回所有已注册 skill 的名称+描述+路径，注入 system prompt 后引导 LLM 按需 `Read` skill 目录下的 `SKILL.md`。这种「延迟加载」范式避免了将所有 skill 内容一次性塞入上下文。

---

## 局限性

**1. 工具组激活状态为进程内单例**

`Toolkit.state_dict()` 只序列化 `active_groups` 列表，没有工具本身的序列化。跨进程/跨 session 恢复工具状态需要重新注册所有工具再 `load_state_dict()`，在分布式或持久化 Agent 场景下有重复初始化开销。

**2. preset_kwargs 不支持动态修改**

`preset_kwargs` 在 `register_tool_function()` 时一次性设置，没有提供 `update_preset_kwargs()` 接口。需要修改预设参数时必须 `remove_tool_function` + 重新注册，侵入性强。

**3. 工具 schema 完全依赖 docstring 格式**

`_parse_tool_function` 用 `docstring_parser` 解析 docstring，若函数 docstring 缺失或格式不规范（非 Google/Numpy 风格），提取到的 description 为空，LLM 无法正确理解工具用途。没有强制要求或 fallback 机制。

**4. async_execution 实验性，无持久化**

后台异步任务存储在 `self._async_tasks` 和 `self._async_results` 内存 dict 中，进程重启后全部丢失。若 LLM 拿到的 task_id 在重启后查询，返回 `InvalidTaskIdError`，没有优雅的失败恢复路径。

**5. MCP 有状态客户端多实例 LIFO 约束**

文档明确指出多个 `StdIOStatefulClient` 实例必须按 LIFO 顺序 `close()`，否则可能出错（关联 MCP Python SDK issue #577）。这是底层 MCP SDK 的约束透传到上层，使用者需要手动管理关闭顺序，出错时错误信息不够直接。

**6. 中间件链执行存在潜在问题**

`_apply_middlewares` 中构造 `base_handler` 时用 `**kwargs` 解包，但 `func(self, **kwargs)` 实际上 `kwargs` 含 `tool_call` 键，而 `call_tool_function` 签名只接受 `tool_call` 位置参数（不接受额外 kwargs）。中间件扩展 kwargs 时若传入非预期键会直接报错，扩展性存在隐患。代码注释中也标注「may include additional context in future versions」，尚未稳定。

**7. Agent Skill 无版本管理**

`register_agent_skill` 只读取 `name` 和 `description`，没有版本字段。同名 skill 重复注册会 `raise ValueError`，但不支持版本升级（覆盖已有 skill 需要先 `remove_agent_skill` 再重新注册）。

---

## 来源

- 源码版本：0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12
- 分析深度：源码级
- 核心文件：`src/agentscope/tool/_toolkit.py`（1700+ 行）、`src/agentscope/tool/_types.py`、`src/agentscope/tool/_response.py`、`src/agentscope/tool/_async_wrapper.py`、`src/agentscope/mcp/`（6 个文件）
