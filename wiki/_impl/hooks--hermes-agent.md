---
title: "Hooks — Hermes Agent"
category: L2
parent: "[[hooks]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的 Hooks 系统采用**双轨并行架构**：一套是 `AIAgent` 类上的**回调参数体系**（6+个 callable 字段），另一套是 `PluginManager` 的**生命周期 Hook 体系**（10个具名 hook 事件）。两套系统服务于不同的场景：回调体系面向平台集成层（CLI、Gateway、ACP），由 callsite 在构建 AIAgent 时注入；Plugin hook 体系面向第三方插件，通过 `register_hook()` 注册到全局单例 `PluginManager` 中。两者在 `run_agent.py` 的同一主循环内被分开调用，并不互通。

## 架构分析

### 回调体系 — AIAgent 的 6+个 Callback 字段

`AIAgent.__init__`（`run_agent.py` line 433）接受以下可调用参数，全部保存为实例属性：

| 回调字段 | 签名 | 触发时机 |
|---------|------|---------|
| `tool_progress_callback` | `(event_type, name, preview, args, **kwargs)` | 工具开始/完成时，以及 subagent 推理转发时 |
| `tool_start_callback` | `(tool_call_id, name, args)` | 工具开始时（独立于 progress，携带 tool_call_id） |
| `tool_complete_callback` | `(tool_call_id, name, args, result)` | 工具执行完成后 |
| `thinking_callback` | `(text: str)` | API 调用期间的"正在思考"动画文本更新 |
| `reasoning_callback` | `(text: str)` | 流式推理 token（OpenRouter extended thinking）到达时 |
| `stream_delta_callback` | `(text: str \| None)` | 流式文本 token；`None` 表示工具轮次边界 |
| `tool_gen_callback` | `(tool_name: str)` | 模型开始生成工具调用参数时（流式中） |
| `step_callback` | `(api_call_count: int, prev_tools: list)` | 每轮 LLM 调用前（携带上一轮工具结果） |
| `status_callback` | `(event_type: str, message: str)` | context pressure 等状态通知 |

**注意**：`tool_progress_callback` 是一个多路复用回调，以 `event_type` 字符串区分事件（`"tool.started"`、`"tool.completed"`、`"reasoning.available"`、`"_thinking"`），而非为每种事件设独立的回调。

### tool_progress_callback 的 event_type 语义

`run_agent.py` 中 `tool_progress_callback` 在三处以不同 `event_type` 触发：

**1. 工具启动时**（sequential path line 6299；concurrent path line 6103）：
```python
self.tool_progress_callback("tool.started", name, preview, args)
```
`preview` 由 `_build_tool_preview(name, args)` 生成，是从工具参数中提取的人类可读摘要（如 terminal 工具提取 command 字段的前 N 个字符）。

**2. 工具完成时**（sequential path line 6495；concurrent path line 6175）：
```python
self.tool_progress_callback("tool.completed", function_name, None, None,
                            duration=tool_duration, is_error=is_error)
```
完成事件不携带 preview 和 args，但通过 `**kwargs` 传递 `duration` 和 `is_error`。

**3. subagent 思考转发**（line 8516）：
```python
# subagent（delegate_depth > 0）：把第一行思考转发给父 agent
self.tool_progress_callback("_thinking", first_line)
# 顶层 agent：发送结构化推理
self.tool_progress_callback("reasoning.available", "_thinking", think_text[:500], None)
```
这是 subagent 将自身的推理内容透传给父 agent 显示层的机制。

### step_callback — 每轮 LLM 调用前触发

`step_callback` 在主循环（`run_agent.py` line 7131）中每次 `while api_call_count < self.max_iterations` 迭代的**最开始**触发，早于 API 调用本身：

```python
if self.step_callback is not None:
    prev_tools = []  # 从 messages 中反向查找上一个 assistant turn 的工具调用
    # 构建 prev_tools: [{"name": tc_name, "result": result_content}, ...]
    self.step_callback(api_call_count, prev_tools)
```

`prev_tools` 是通过遍历 `messages` 列表找到最近一个 assistant message 的 `tool_calls`，并从后续 `role=tool` 消息中提取 `content` 来组装的，不是实时数据，而是回顾上一轮已完成的工具执行记录。第一轮调用时 `prev_tools` 为空列表。

### stream_delta_callback — None 的特殊语义

`stream_delta_callback` 接收流式文本 token，但 `None` 值携带特殊语义（`run_agent.py` line 8800）：

```python
# 工具执行前，向 display callback 发送 None 信号
if self.stream_delta_callback:
    self.stream_delta_callback(None)
```

这个 `None` 信号通知下游"当前流已结束，工具执行即将开始"。CLI 的 `_stream_delta` 实现收到 `None` 后调用 `_flush_stream()` + `_reset_stream_state()` 来关闭当前渲染的文本框。注意 TTS 回调（`_stream_callback`）不接收 `None`，只有 display callback 接收，通过 `_fire_stream_delta` 中的分支保护实现。

### thinking_callback — 双重用途

`thinking_callback` 并不只用于流式推理内容。它在 API 调用**等待期间**被调用来更新 TUI 加载动画（`run_agent.py` line 7269）：

```python
if self.thinking_callback:
    self.thinking_callback(f"{face} {verb}...")  # e.g. "🧠 pondering..."
```

当 API 响应到达或发生错误时，发送空字符串 `""` 来清除动画（line 7346）：

```python
if self.thinking_callback:
    self.thinking_callback("")
```

而真正的扩展推理（extended reasoning token），走的是独立的 `reasoning_callback`（`_fire_reasoning_delta`，line 4250）。两者在 CLI 中被分开处理：`thinking_callback` 更新 spinner 文字，`reasoning_callback` 渲染推理内容块。

### Plugin Hook 体系 — 10个生命周期事件

`hermes_cli/plugins.py` 中定义了 `VALID_HOOKS` 集合，共 10 个：

```python
VALID_HOOKS = {
    "pre_tool_call",
    "post_tool_call",
    "pre_llm_call",
    "post_llm_call",
    "pre_api_request",
    "post_api_request",
    "on_session_start",
    "on_session_end",
    "on_session_finalize",
    "on_session_reset",
}
```

这套系统通过 `PluginContext.register_hook(hook_name, callback)` 注册，由全局单例 `PluginManager.invoke_hook(name, **kwargs)` 调用。各 hook 的调用方采用 `try/except` 隔离，单个插件崩溃不影响其他插件或主循环。

`pre_api_request` 和 `post_api_request` 在 `run_agent.py` 的主循环中（line 7306 和 8483）被直接调用，传递详细的 API 请求/响应元数据（model、provider、token 数量、duration 等）。

`pre_llm_call` 支持特殊返回值：插件返回字符串或 `{"context": "..."}` 字典时，该内容会被注入当前轮次的 user message 中。注意这是 API-call-time 注入（临时），不会持久化到 session DB，也不破坏 system prompt 缓存前缀（context 只进 user message）。

## 关键代码路径

- `run_agent.py` line 461–468 — `AIAgent.__init__` 中全部 callback 参数声明
- `run_agent.py` line 592–602 — callback 字段赋值（`self.tool_progress_callback = ...`）
- `run_agent.py` line 6299–6304 — sequential path：工具启动时调用 `tool_progress_callback`
- `run_agent.py` line 6306–6310 — sequential path：工具启动时调用 `tool_start_callback`
- `run_agent.py` line 6495–6502 — sequential path：工具完成时调用 `tool_progress_callback`
- `run_agent.py` line 6511–6515 — sequential path：工具完成时调用 `tool_complete_callback`
- `run_agent.py` line 6103–6115 — concurrent path：并行工具启动前的 progress + start callbacks
- `run_agent.py` line 6175–6207 — concurrent path：并行工具完成后的 progress + complete callbacks
- `run_agent.py` line 7130–7155 — `step_callback` 触发（每轮 LLM 调用前）
- `run_agent.py` line 7266–7269 / 7345–7372 — `thinking_callback` 触发（API 等待期动画）
- `run_agent.py` line 8516–8534 — `tool_progress_callback` 传递 subagent 推理内容
- `run_agent.py` line 8800–8804 — `stream_delta_callback(None)` 工具轮次边界信号
- `run_agent.py` line 4234–4273 — `_fire_stream_delta` / `_fire_reasoning_delta` / `_fire_tool_gen_started` 内部封装
- `hermes_cli/plugins.py` line 55–66 — `VALID_HOOKS` 定义
- `hermes_cli/plugins.py` line 218–233 — `PluginContext.register_hook()`
- `hermes_cli/plugins.py` line 468–502 — `PluginManager.invoke_hook()`（带 try/except 隔离）
- `hermes_cli/callbacks.py` — `clarify_callback`、`approval_callback`、`prompt_for_secret` 的 TUI 实现
- `acp_adapter/events.py` — ACP callback 工厂函数（`make_tool_progress_cb`、`make_thinking_cb`、`make_step_cb`）
- `gateway/run.py` line 6330–6391 — gateway `progress_callback` 实现（queue + 进度消息积累）
- `gateway/run.py` line 6526–6549 — gateway `_step_callback_sync`（sync → async bridge）

## 各 callsite 的 callback 接入策略

四个平台入口对同一套 callback 接口采用完全不同的实现策略，这是 Hermes 回调体系设计价值的集中体现：

### CLI（`cli.py`）

CLI 在构建 `AIAgent` 时传入完整的 callback 集合（line 2459–2471）：

- `thinking_callback = self._on_thinking` → 更新 prompt_toolkit 的 spinner 文本（`self._spinner_text`），触发 TUI `invalidate()`
- `tool_progress_callback = self._on_tool_progress` → `event_type == "tool.started"` 时更新 spinner label 并（voice mode 下）播放音频 beep
- `tool_start_callback = self._on_tool_start`（仅在 `inline_diffs_enabled` 时）→ 调用 `capture_local_edit_snapshot()` 截取文件编辑前状态
- `tool_complete_callback = self._on_tool_complete`（仅在 `inline_diffs_enabled` 时）→ 调用 `render_edit_diff_with_delta()` 展示 inline diff
- `stream_delta_callback = self._stream_delta`（仅在 `streaming_enabled` 时）→ 行缓冲流式渲染，处理 reasoning XML tag 过滤
- `tool_gen_callback = self._on_tool_gen_start`（仅在 `streaming_enabled` 时）→ 关闭流式文本框，打印"preparing tool_name..."状态行
- `clarify_callback = self._clarify_callback` → 通过 `queue.Queue` + TUI `invalidate()` 实现跨线程的用户交互弹窗，含 120s 超时

### Gateway（`gateway/run.py`）

Gateway 以**每轮消息动态绑定**的方式设置 callback（而非在 `AIAgent.__init__` 中传入，因为 agent 实例被缓存跨消息复用）：

```python
# Per-message state — callbacks change every turn
agent.tool_progress_callback = progress_callback if tool_progress_enabled else None
agent.step_callback = _step_callback_sync if _hooks_ref.loaded_hooks else None
agent.stream_delta_callback = _stream_delta_cb
agent.status_callback = _status_callback_sync
```

`progress_callback` 将 `tool.started` 事件格式化为消息字符串放入 `queue.Queue`，由异步 `send_progress_messages()` task 消费，通过平台 adapter 的 `edit_message()` API 实时编辑进度消息（每 1.5 秒节流，避免 Telegram flood control）。还实现了去重逻辑：相同消息出现时发送 `("__dedup__", msg, count)` 元组在消息末尾追加 `×N`。

`_step_callback_sync` 是 sync→async 的桥接函数，通过 `asyncio.run_coroutine_threadsafe` 向 `hooks.emit("agent:step", {...})` 投递事件（供 gateway 的事件钩子系统消费），`step_callback` 仅在 `_hooks_ref.loaded_hooks` 为真时才被注册，避免无用开销。

### ACP Adapter（`acp_adapter/server.py` + `acp_adapter/events.py`）

ACP 使用工厂函数模式，为每个 session 的每次 prompt 调用创建新的回调闭包：

```python
tool_progress_cb = make_tool_progress_cb(conn, session_id, loop, tool_call_ids)
thinking_cb = make_thinking_cb(conn, session_id, loop)
step_cb = make_step_cb(conn, session_id, loop, tool_call_ids)
```

**关键设计**：`tool_progress_callback`（负责 ToolCallStart 事件）和 `step_callback`（负责 ToolCallComplete 事件）通过共享的 `tool_call_ids: Dict[str, Deque[str]]` 字典协作，维护工具调用 ID 的 FIFO 队列，保证同名工具的并行调用也能正确地将 Start 和 Complete 配对。ACP 没有使用 `stream_delta_callback`，streaming 走 `message_callback`（通过 `acp.update_agent_message_text`）。所有回调均通过 `asyncio.run_coroutine_threadsafe` 从 agent 的工作线程向 asyncio 事件循环投递，5s 超时。

### RL 环境（`environments/agent_loop.py`）

`HermesAgentLoop`（RL 训练用的独立 agent loop）**不使用任何 callback**。它采用完全不同的架构：直接操作消息列表，tool 调用通过线程池执行，reasoning 内容被存入 `AgentResult.reasoning_per_turn` 列表由外部消费，不需要实时通知。这使得 RL 环境可以高并发地批量运行（128 worker 线程池），不受 TUI/消息推送等 I/O 的约束。

## 设计亮点

- **多路 event_type 复用单一 callback**：`tool_progress_callback` 以 `event_type` 字符串区分事件类型，避免了参数爆炸（否则需要 tool_started_cb、tool_completed_cb、reasoning_available_cb 等更多字段）。callsite 可以选择性地只响应自己关心的 event_type，其余直接 return
- **stream_delta_callback(None) 作为带外信号**：`None` 值复用了同一个 callback 携带"轮次边界"语义，避免引入额外的 `stream_boundary_callback`。CLI 的实现专门区分 `None` 和空字符串的不同含义
- **Agent 缓存 + per-message callback 动态绑定**（gateway）：将 agent 实例与 callback 实现分离，agent 跨消息被复用（保留 prompt cache），callback 每消息重新绑定（携带当次请求的 queue、loop、chat_id 等上下文），是一个优雅的跨轮次状态管理解决方案
- **ACP 的 tool_call_ids FIFO 协同**：`tool_progress_callback` 和 `step_callback` 通过共享字典协作，巧妙解决了"Start 和 Complete 事件来自不同 callback"的 ID 对齐问题，尤其是并行调用同名工具时不会错位
- **Plugin hook 的 pre_llm_call 上下文注入走 user message 而非 system prompt**：刻意保护 system prompt 不变，从而维持跨轮次的 prompt cache 命中率，是把架构约束（缓存）融入 hook 设计的典型案例

## 局限性

- **callback 字段无类型约束**：所有 callback 参数声明为 `callable = None`，无 `TypeVar` 或 `Protocol` 约束，IDE 无法提示签名，callsite 若传入签名不匹配的函数，运行时才会在 `try/except` 中静默失败（只记录 debug log，不报错）
- **tool_progress_callback 的多路复用增加理解成本**：`event_type` 设计实际上把多个独立事件混在一个 callback 中，每个 callsite 都要自行过滤（如 gateway 和 CLI 的实现都有 `if event_type != "tool.started": return`），不如分开的事件回调自描述
- **Plugin hook 体系与回调体系完全隔离**：两套系统没有桥接。Plugin 的 `pre_tool_call` / `post_tool_call` hook 并没有实际被调用（在 `run_agent.py` 的工具执行路径中找不到 `invoke_hook("pre_tool_call")` 的调用点），说明 Plugin hook 的某些 hook 可能是占位定义，尚未完全落地
- **step_callback 的语义偏移**：名为"step"，实际触发时机是"下一轮 LLM 调用之前"，携带的是上一轮的工具结果，而非当前 step 的开始状态，这在 gateway 的 `agent:step` 事件处理中需要特别注意
- **RL 环境不继承 callback 架构**：`HermesAgentLoop` 是一套独立实现，不复用 `AIAgent` 的 callback 体系，导致两套代码分叉维护，callback 方面的改进不会自动同步到 RL 路径

## 来源

- 源码版本：hermes-agent 0.16.0
- 分析深度：源码级
- 主要文件：`run_agent.py`、`cli.py`、`gateway/run.py`、`acp_adapter/server.py`、`acp_adapter/events.py`、`hermes_cli/plugins.py`、`hermes_cli/callbacks.py`、`environments/agent_loop.py`
