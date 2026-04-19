---
title: "channel-remote——SimpleMem"
category: L2
parent: "[[channel-remote]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: channel-remote
created: "2026-04-15"
updated: "2026-04-15"
confidence: medium
---

## 概述

SimpleMem 提供两种远程访问渠道：（1）`cross/api_http.py` 的 FastAPI REST API，将跨会话记忆功能暴露为 HTTP 端点；（2）`cross/api_mcp.py` + `MCP/` 的 MCP 协议接口，供兼容 MCP 的客户端（Claude Desktop、Cursor 等）接入。两种渠道共享同一个 `CrossSessionOrchestrator` 后端，通过不同协议适配层提供一致的记忆服务。

## 架构分析

### HTTP REST API（cross/api_http.py）

基于 FastAPI 实现，提供以下端点：

| 端点 | 方法 | 功能 |
|------|------|------|
| `/sessions` | POST | 开始新会话（StartSessionRequest） |
| `/sessions/{id}/messages` | POST | 记录消息事件 |
| `/sessions/{id}/tool-uses` | POST | 记录工具调用事件 |
| `/sessions/{id}/stop` | POST | 结束并持久化会话 |
| `/sessions/{id}` | DELETE | 释放会话资源 |
| `/search` | POST | 语义搜索记忆 |
| `/health` | GET | 健康检查 |

使用 Pydantic 模型做请求/响应验证，CORS middleware 允许前端跨域访问。

### MCP 协议接口（MCP/ + cross/api_mcp.py）

MCP server 作为独立进程，通过 stdio 或 HTTP/SSE transport 与 MCP 客户端通信：

```
MCP Client (Claude Desktop/Cursor)
    ↓ MCP protocol (stdio/SSE)
MCP Server (MCP/server/run.py)
    ↓ Python function calls
MCPToolRegistry (cross/api_mcp.py)
    ↓ async/sync dispatch
CrossSessionOrchestrator (cross/orchestrator.py)
```

### 部署选项

- **Docker Compose**：`docker-compose.yml` 一键启动 HTTP API server + 可选 MCP server
- **本地开发**：`MCP/run.sh` 脚本启动 MCP server（stdio 模式）
- **嵌入式**：直接 import `CrossSessionOrchestrator` 嵌入现有 Python 应用

## 关键代码路径

### FastAPI 应用创建（create_app）

```python
# cross/api_http.py
def create_app(project: str = "default", db_path: str = "./memory.db") -> FastAPI:
    app = FastAPI(title="SimpleMem Cross-Session API", version="0.1.0")
    app.add_middleware(CORSMiddleware, allow_origins=["*"], ...)
    
    orchestrator = CrossSessionOrchestrator(
        db_path=db_path,
        project=project,
    )
    
    router = create_cross_router(orchestrator)
    app.include_router(router, prefix="/cross")
    
    @app.get("/health")
    async def health():
        return {"status": "ok", "timestamp": time.time()}
    
    return app

def create_cross_router(orchestrator) -> APIRouter:
    router = APIRouter()
    # 注册所有端点，闭包捕获 orchestrator
    @router.post("/sessions", response_model=StartSessionResponse)
    async def start_session(req: StartSessionRequest):
        result = await orchestrator.session_start(
            tenant_id=req.tenant_id,
            content_session_id=req.content_session_id,
            project=req.project,
            user_prompt=req.user_prompt,
        )
        return StartSessionResponse(
            memory_session_id=result.memory_session_id,
            context=result.context_bundle.formatted_context if result.context_bundle else "",
            context_tokens=result.context_bundle.total_tokens if result.context_bundle else 0,
        )
    return router
```

### HTTP 响应模型（Pydantic）

```python
class StartSessionResponse(BaseModel):
    memory_session_id: str         # UUID, 后续调用使用
    context: str                   # 历史 context 文本（可为空）
    context_tokens: int            # context token 估算数

class StopSessionResponse(BaseModel):
    memory_session_id: str
    observations_count: int        # 提取的 observation 数量
    summary_generated: bool
    entries_stored: int

class SearchEntry(BaseModel):
    text: str
    score: float                   # 相似度分数 [0, 1]
```

### MCP Server 启动（MCP/server/run.py）

```python
# MCP/server/run.py
import mcp  # MCP SDK
from cross.api_mcp import create_mcp_tools
from cross.orchestrator import CrossSessionOrchestrator

orchestrator = CrossSessionOrchestrator(...)
registry = create_mcp_tools(orchestrator)

server = mcp.Server("simplemem-cross")

@server.list_tools()
async def list_tools():
    return registry.get_tool_definitions()

@server.call_tool()
async def call_tool(name: str, arguments: dict):
    return await registry.call_tool(name, arguments)

mcp.run(server, transport="stdio")  # 或 "sse"
```

## 设计亮点

1. **双协议覆盖**：同一 orchestrator 后端同时支持 REST（通用 HTTP 客户端）和 MCP（IDE/agent 专用协议），不同集成场景无需修改核心逻辑。

2. **可挂载路由（create_cross_router）**：HTTP API 以 FastAPI Router 形式提供，可直接 `include_router` 挂载到用户现有 FastAPI 应用，零侵入集成。

3. **Docker Compose 一键部署**：`docker-compose.yml` 将 memory server 容器化，`volumes` 挂载持久化 db，适合快速原型和轻量生产部署。

4. **CORS 开放策略 + 前端预留**：`allow_origins=["*"]` + MCP 的 `frontend/` 目录表明系统预留了 Web UI 集成路径，不仅是 headless API。

## 局限性

1. **HTTP API 无认证**：`create_app` 没有 API Key、JWT 或 OAuth 中间件，直接暴露到网络存在安全风险，生产部署需在 API Gateway 层另加认证。

2. **MCP Server 无连接复用**：每次 MCP client 重连都创建新的 orchestrator 实例，无法保持 `_active_sessions` 状态，长会话跨连接断续场景不支持。

3. **HTTP API 无分页**：`/search` 端点返回最多 100 条结果（inputSchema 中 top_k 上限 100），无游标/分页，大量结果场景性能不佳。

4. **MCP server transport 文档不完整**：`MCP/README.md` 较简略，stdio vs SSE transport 的选择、Claude Desktop 的配置格式（claude_desktop_config.json）未明确，集成需参考 MCP 协议外部文档。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
