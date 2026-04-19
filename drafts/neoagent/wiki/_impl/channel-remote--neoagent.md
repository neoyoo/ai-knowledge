---
title: "channel-remote — neoagent"
category: L2
parent: "[[channel-remote]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: channel-remote
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的 channel 系统定义了 `Channel` ABC（`channels/base.py`，4 个抽象方法：`start/stop/serve_forever/__init__`）+ 唯一具体实现 `FastAPIChannel`（357 行）。`FastAPIChannel` 暴露三个 HTTP endpoint：`GET /v1/health`、`POST /v1/run`（同步返回完整 `ConversationResult`）、`POST /v1/run/stream`（SSE 流）。架构亮点：**每请求 stateless session**（无跨请求记忆）、**`asyncio.Queue` 桥接同步 EventBus → async generator**（让 sync handler 产生的事件流过 async SSE）、**`asyncio.Lock` 单请求串行**（避免同一 NeoAgent 并发造成 ToolRegistry 状态竞争）、**Bearer API key + CORS 可选中间件**（生产级安全）。MCP 的 `StdioTransport`（`mcp/transport.py`）是 channel 体系外的另一种"远程/子进程"通信，独立实现。

## 架构分析

### Channel ABC

```python
class Channel(ABC):
    _agent: "NeoAgent"

    def __init__(self, agent: "NeoAgent") -> None:
        self._agent = agent

    @abstractmethod
    async def start(self) -> None: ...    # bind port / connect broker
    @abstractmethod
    async def stop(self) -> None: ...      # graceful shutdown
    @abstractmethod
    async def serve_forever(self) -> None: ...  # block until stop()
```

注释明写 `stop() may be called concurrently from a signal handler or another asyncio task`——所有实现必须处理并发停止。

### FastAPIChannel 三端点

#### `GET /v1/health`
```python
@app.get("/v1/health")
async def health():
    return {"status": "ok"}
```
**跳过 API key 校验**（auth middleware 显式 if 判断 `request.url.path == "/v1/health"` 直通）——负载均衡探活不需要凭证。

#### `POST /v1/run`
```python
async def run(req: RunRequest):
    msgs = [Message(role=m.role, content=m.content) for m in req.messages]
    async with self._run_lock:
        result = await self._agent.run(messages=msgs, max_turns=req.max_turns)
    return RunResponse.from_result(result)
```
- **`self._run_lock = asyncio.Lock()`**：全 channel 实例一把锁，所有 `/v1/run` 请求串行化——防止同一 NeoAgent 被两个请求并发调用导致 ToolRegistry / PromptBuilder 状态污染。
- **Stateless**：每次请求构造 messages 直接送 `agent.run()`，不经过 Session 持久化（`NeoAgent.run()` 无 session 参数时创建临时 session）——无跨请求记忆。
- **`RunResponse.from_result` 序列化**：提取每个 Turn 的 text（`TextBlock.text` 拼接）+ tool_calls 数组（仅 name + id），不返回 tool_result 原文（避免响应体膨胀）。

#### `POST /v1/run/stream`
SSE 流式响应：
```python
async def _sse_generator(self, messages, max_turns):
    queue: asyncio.Queue[Event | BaseException | None] = asyncio.Queue()
    
    # 注册 sync handlers 把事件推到 queue
    handlers = {}
    for event_type in _STREAM_EVENT_TYPES:
        def _make_handler():
            def _handler(ev: Event) -> None:
                queue.put_nowait(ev)
            return _handler
        h = _make_handler()
        handlers[event_type] = h
        self._agent.event_bus.subscribe(event_type, h)
    
    # 启动 agent task
    async def _run():
        async with self._run_lock:
            return await self._agent.run(messages=messages, max_turns=max_turns)
    agent_task = asyncio.create_task(_run())
    
    def _on_done(fut):
        if fut.cancelled():
            queue.put_nowait(asyncio.CancelledError(...))
        else:
            queue.put_nowait(fut.exception())  # None on success
    agent_task.add_done_callback(_on_done)
    
    try:
        while True:
            item = await queue.get()
            if item is None:
                yield f"data: {json.dumps({'type': 'done', 'reason': result.reason})}\n\n"
                break
            elif isinstance(item, BaseException):
                yield f"data: {json.dumps({'type': 'error', 'message': str(item)})}\n\n"
                break
            else:
                yield f"data: {_event_to_sse_data(item)}\n\n"
    finally:
        # 确保 unsubscribe，防止 handler 泄漏
        for event_type, handler in handlers.items():
            self._agent.event_bus.unsubscribe(event_type, handler)
        if not agent_task.done():
            agent_task.cancel()
            try: await agent_task
            except (asyncio.CancelledError, Exception): pass
```

**`_STREAM_EVENT_TYPES`** = ToolCallEvent / ToolResultEvent / TurnCompleteEvent / ProviderResponseEvent——只推这 4 种（不是全部 16 种），降低 SSE 带宽。每种 event 有定制化 SSE payload（`_event_to_sse_data`）：

```json
{"type": "tool_call", "name": "...", "call_id": "...", "input": {...}}
{"type": "tool_result", "name": "...", "call_id": "...", "output": "...", "is_error": false}
{"type": "turn_complete", "turn_index": N, "stop_reason": "...", "tool_call_count": M}
{"type": "response", "stop_reason": "...", "input_tokens": N, "output_tokens": M, "turn": K}
```

**响应头** `X-Accel-Buffering: no` + `Cache-Control: no-cache`——禁 nginx/cloudflare 缓冲，保证浏览器实时接收。

### 鉴权 + CORS 中间件

```python
if self._api_key is not None:
    @app.middleware("http")
    async def auth_middleware(request, call_next):
        if request.url.path == "/v1/health":
            return await call_next(request)
        # Bearer check
        auth = request.headers.get("Authorization", "")
        if not auth.startswith("Bearer ") or auth[7:] != self._api_key:
            return JSONResponse(status_code=401, content={"detail": "Invalid or missing API key"})
        return await call_next(request)

if self._cors_origins is not None:
    app.add_middleware(
        CORSMiddleware,
        allow_origins=self._cors_origins,
        allow_methods=["GET", "POST"],
        allow_headers=["Authorization", "Content-Type"],
        allow_credentials=False,
    )
```

`allow_credentials=False` 是刻意的——和 `allow_origins=["*"]` 配合时符合 CORS 安全规范；要求传 cookie 的场景必须把 origin 写死并改为 True。

### Lazy import 设计

FastAPI 和 uvicorn 通过 `_get_fastapi()` / `_get_uvicorn()` 延迟导入，捕获 `ImportError` 给出清晰错误信息：`"FastAPIChannel requires the 'fastapi' extra: pip install neoagent[fastapi]"`——核心包不强依赖 fastapi，保持轻量。

### MCP StdioTransport 作为另一种 remote channel

虽然在 `mcp/` 目录而非 `channels/`，但 `StdioTransport`（`transport.py` 149 行）本质是一种"远程"通信抽象：

- `MCPTransport` ABC：`connect/send/receive/close` 4 方法
- `StdioTransport` 子进程 stdio 实现：
  - **Env 白名单**：`_DEFAULT_ENV_WHITELIST = {"PATH","HOME","USER","LOGNAME","LANG","LC_ALL","TERM","TMPDIR","TZ","SHELL"}`——**只把 9 个安全环境变量透传给 MCP 子进程**，防止 API key 等泄漏；`_safe_env()` 把用户通过 `env=` 参数传的 extras 合并进去（用户显式声明的视为安全）。
  - **Stderr drain task**：`_drain_stderr` 后台 async task 持续 readline，防止 MCP server 写 stderr 但无人读导致 pipe buffer 满而进程死锁。
  - **Close 流程**：cancel stderr task 并 await → `proc.terminate()` + 5s 超时 → `proc.kill()` 兜底。
  - **Receive EOF 处理**：`line_bytes = b""` 抛 `ConnectionError` 而非返回空 dict，让上层 `MCPClient` 明确感知断连。

### 关键代码路径

- `neoagent/channels/base.py:11-45` — Channel ABC
- `neoagent/channels/fastapi_channel.py:134-203` — FastAPIChannel 类 + start/stop/serve_forever
- `neoagent/channels/fastapi_channel.py:208-282` — `_build_app`（auth/CORS 中间件 + 端点注册）
- `neoagent/channels/fastapi_channel.py:296-357` — `_sse_generator`（queue bridge + handler 注册/清理）
- `neoagent/channels/fastapi_channel.py:260-265` — `run.__annotations__ = {...}` 强注入 FastAPI 类型（闭包局部 class 无法解析 string annotation）
- `neoagent/mcp/transport.py:46-149` — StdioTransport
- `neoagent/mcp/transport.py:12-15` — 环境变量白名单

## 设计亮点

- **asyncio.Queue 桥接同步 EventBus 到 async generator**：EventBus handler 是同步 callback，但 SSE 需要 `async for` 语义。用 `queue.put_nowait` + `await queue.get()` 的 producer-consumer 模式桥接——不修改 EventBus 的同步设计，也不阻塞 sse generator。这是 async Python 里常见但实现容易出错的模式（错误处理 / task done 回调 / handler 泄漏），neoagent 把三个边缘都处理到位。
- **`_on_done` 把 exception 也 put 进 queue**：让 generator 的单一循环 `while True: item = await queue.get()` 同时处理正常完成、异常、和正常事件——消除"await queue 等超时 + await task 等完成"的双 select 复杂度。
- **`finally` 块双重清理**：unsubscribe all handlers + cancel agent task——防止客户端提前断连时 handler 和 agent 泄漏。多数 SSE 实现在这里有 bug。
- **`/v1/health` 跳过鉴权**：显式白名单路径设计，负载均衡不需要 API key 也能探活。
- **`asyncio.Lock` 单请求串行**：这是务实的选择——让同一 NeoAgent 实例安全地服务多个 HTTP 请求，代价是吞吐受限。想水平扩展就创建多个 channel 实例 + 每个 bind 不同 NeoAgent。
- **Lazy FastAPI import**：核心包无 FastAPI/uvicorn 依赖，extras 机制让"只做 CLI 使用"的场景不拖进额外 30+ 个依赖；import error 给出精确 pip 命令指引。
- **StdioTransport 的 env 白名单**：子进程继承的 env 只有 9 个安全键（无 API_KEY/AWS_SECRET 等）；用户显式 extras 才添加——这是"secure by default"的环境变量隔离，比大多数开源框架直接 `env=os.environ.copy()` 安全得多。
- **Stderr drain task**：防止 MCP 子进程写 stderr 死锁，是工程细节但极易忘。

## 局限性

- **唯一 Channel 实现是 FastAPI**：没有 WebSocket / gRPC / MQ 实现，要接入其他传输必须自己继承 Channel 写。
- **每请求 stateless**：`/v1/run` 无法跨请求共享 session——如果想要 "用户 A 的第二次请求继续上次对话"，必须在 channel 外自己管 Session 池 + 每请求显式 load + save。
- **`asyncio.Lock` 串行化限吞吐**：多用户场景下单 agent 实例处理速度是天花板；真正生产要靠多 channel 实例 + 前端负载均衡。
- **SSE event 粒度粗**：只推 4 种 event 类型，无法流式推送 LLM 每个 TextBlock token（provider 的 streaming 能力未 expose 到 channel 层）——前端无法做 "打字机效果"。
- **无请求级 request_id 追踪**：SSE 中各种 event 没有 "这是哪个请求的" 关联——高并发下日志追踪困难（LDR 头不会传到 event 里）。
- **MCP StdioTransport 无重连**：MCP server 崩溃后，`receive()` 抛 ConnectionError，`MCPClient` 无 reconnect 逻辑——需要重启整个 NeoAgent。
- **MCP SseTransport 留空**：`transport.py` 注释 "Reserved for future: SseTransport (HTTP+SSE), not implemented"——远程 MCP server 只能走 stdio，不能接入已有 MCP HTTP 服务。
- **API key 校验是固定字符串比较**：`auth[7:] != expected_key`——无 HMAC 签名、无时序攻击防护（`!=` 在字节级提前退出）；生产安全可用但不是银弹。
- **CORS `allow_credentials=False`** 硬编码：用户想要 cookie 传递必须改源码；API 不 expose 这个字段。
- **FastAPIChannel 不处理 reverse proxy 的 X-Forwarded-For**：rate limit / audit 看到的是 proxy IP 而非 client IP——需要用户自己加 middleware。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/channels/base.py`, `neoagent/channels/fastapi_channel.py`, `neoagent/mcp/transport.py`
