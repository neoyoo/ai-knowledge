---
title: "mcp-skills——SimpleMem"
category: L2
parent: "[[mcp-skills]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: mcp-skills
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

SimpleMem 提供三种协议集成方式：（1）`cross/api_mcp.py` 的 `MCPToolRegistry` 将跨会话记忆功能包装为 8 个标准 MCP tool，供 MCP 客户端（Claude Desktop/Cursor 等）直接调用；（2）`MCP/` 目录包含独立的 MCP 服务端实现；（3）`SKILL/simplemem-skill/` 是 Claude Code skill 封装，供 Claude Code skill 体系调用。三种方式覆盖了 MCP 协议集成的完整路径。

## 架构分析

### MCPToolRegistry（cross/api_mcp.py）

8 个 MCP tool，覆盖完整 session 生命周期：

| Tool Name | 功能 |
|-----------|------|
| `cross_session_start` | 开始新会话，返回 memory_session_id + 历史 context |
| `cross_session_message` | 记录消息事件 |
| `cross_session_tool_use` | 记录工具调用事件 |
| `cross_session_stop` | 结束并持久化会话（触发 observation 提取） |
| `cross_session_end` | 释放会话资源 |
| `cross_session_search` | 语义搜索历史记忆 |
| `cross_session_context` | 获取适合注入 system prompt 的 context 文本 |
| `cross_session_stats` | 查询记忆系统统计信息 |

### MCP Server（MCP/ 目录）

```
MCP/
├── config/          # 服务端配置
├── frontend/        # 可选 Web 前端
├── server/          # FastAPI/MCP server 实现
│   ├── run.py       # 服务启动入口
│   └── register.py  # tool 注册
├── reference/       # 参考实现
└── requirements.txt
```

MCP server 作为独立进程运行，通过 HTTP/SSE 或 stdio transport 与 MCP 客户端通信，内部调用 `MCPToolRegistry` 派发工具调用。

### Claude Code Skill（SKILL/simplemem-skill/）

将 SimpleMem 功能封装为 Claude Code skill，使用户在 Claude Code 中通过 `/simplmem` 命令触发记忆操作（查询历史、保存当前会话等）。

## 关键代码路径

### MCP Tool 定义与分发

```python
# cross/api_mcp.py

# Tool schema 定义（MCP tools/list 响应）
MCPToolRegistry.get_tool_definitions() -> List[Dict]:
    return [
        {
            "name": "cross_session_start",
            "description": "Start a new cross-session memory session...",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "tenant_id": {"type": "string"},
                    "content_session_id": {"type": "string"},
                    "project": {"type": "string"},
                    "user_prompt": {"type": "string"},
                },
                "required": ["tenant_id", "content_session_id", "project"],
            },
        },
        # ... 7 more tools
    ]

# 工具调用分发（MCP tools/call）
async def call_tool(self, name: str, arguments: Dict) -> Dict:
    handler = self._tool_map.get(name)  # name → bound method
    if handler is None:
        return {"error": f"Unknown tool: {name}"}
    return await handler(**arguments)
```

### 核心 Tool 实现（cross_session_start）

```python
async def cross_session_start(
    self,
    tenant_id: str,
    content_session_id: str,
    project: str,
    user_prompt: Optional[str] = None,
) -> Dict[str, Any]:
    result = await _await_if_coro(
        self._orchestrator.session_start(
            tenant_id=tenant_id,
            content_session_id=content_session_id,
            project=project,
            user_prompt=user_prompt,
        )
    )
    return _normalise_result(result, fallback_key="session")
    # 返回: {"memory_session_id": "uuid", "context": "...", "context_tokens": N}
```

### 工厂函数（集成入口）

```python
# 集成到现有 MCP server
registry = create_mcp_tools(orchestrator)
tool_defs = registry.get_tool_definitions()   # → tools/list
result = await registry.call_tool("cross_session_stats", {})  # → tools/call
```

### 结果规范化（_normalise_result）

```python
def _normalise_result(result, *, fallback_key: str) -> Dict:
    if isinstance(result, dict): return result          # 已是 dict
    if hasattr(result, "model_dump"): return result.model_dump()   # Pydantic v2
    if hasattr(result, "dict"): return result.dict()    # Pydantic v1
    if hasattr(result, "__dataclass_fields__"): return result.__dict__  # dataclass
    return {fallback_key: result}                       # fallback
```

## 设计亮点

1. **完整 Session 生命周期 MCP 覆盖**：8 个 tool 精确对应 agent session 的每个阶段（start/message/tool_use/stop/end），agent 框架无需自行管理状态，全部委托给 MCP tool。

2. **Duck-typed Orchestrator**：`_resolve_method()` 在多个候选方法名之间按优先级查找（`session_start` / `start_session`），允许不同版本或自定义 orchestrator 实现只要有匹配方法就能工作，向后兼容性强。

3. **同步/异步透明适配**：`_await_if_coro()` 自动检测返回值是否为 coroutine 并 await，使 MCPToolRegistry 对同步和异步 orchestrator 均透明工作，无需为两种情况分别实现。

4. **JSON Schema 完整类型注解**：每个 tool 的 `inputSchema` 包含详细 description 和类型约束（pattern 验证 role 字段），客户端（Claude Desktop 等）可据此自动生成 UI 和验证参数。

5. **三层集成路径**：MCP tool（API 层）+ MCP server（独立进程）+ Claude Code skill（IDE 层），覆盖从本地 IDE 到远程服务的完整部署场景。

## 局限性

1. **MCP Server 实现文档缺失**：`MCP/server/` 目录缺少详细 README，`run.py` 的 transport 模式（stdio vs HTTP）和认证配置不明确，生产部署依赖外部文档。

2. **无 Tool 版本控制**：`get_tool_definitions()` 返回的 schema 无版本字段，工具接口变更时客户端无法感知，向后兼容性需人工维护。

3. **`cross_session_search` 无租户过滤**：`cross_session_search` tool 的 inputSchema 没有 `tenant_id` 参数，搜索默认跨所有租户，多租户部署有数据泄露风险（需通过 orchestrator 配置全局 tenant_id 绕过）。

4. **Claude Code Skill 未集成 Core 文本版**：`SKILL/simplemem-skill/` 仅封装跨会话 cross/ 功能，不能直接访问 `SimpleMemSystem`（文本版）的 `add_dialogue`/`ask` 接口。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
