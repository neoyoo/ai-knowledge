---
title: "query-loop——agentscope"
category: L2
parent: "[[query-loop]]"
source: agentscope
source_version: "v2.0.1-11-g0e5418e8"
concept: "query-loop"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的 query loop 已从旧版 `AgentBase` / `ReActAgent` / metaclass hook / `pipeline` 结构，收敛到单个统一 `Agent` 类：`reply_stream()` 输出事件流，`_reply_impl()` 驱动 reasoning-acting loop，middleware 以 onion/transformer 方式包裹 reply、reasoning、acting、model call、system prompt 和 context compression。

当前源码中未找到旧页引用的 `src/agentscope/pipeline/`、`MsgHub`、`A2AAgent`、`realtime/`、`tts/` 路径。多 agent/team 现在走 [[multi-agent--agentscope]] 的 session + MessageBus + team tools；跨进程 AgentCard/A2A/Nacos 走 [[agent-registry-discovery--agentscope-java]]。

---

## 当前架构

### 统一 Agent 类

`Agent` 构造函数显式注入：

- `name`
- `system_prompt`
- `model`
- `toolkit`
- `middlewares`
- `state`
- `offloader`
- `model_config`
- `context_config`
- `react_config`

`AgentState` 承载 session_id、summary、context、reply_id、cur_iter、permission_context、tool_context、tasks_context。也就是说 query loop 本身不再是孤立函数，而是围绕可持久化 state 和可注入 toolkit/middleware/offloader 运行。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:94` — `class Agent`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:99` — 构造参数
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/state/_state.py:140` — `AgentState`

### reply_stream / reply 双入口

AgentScope 2.x 有两个公开入口：

- `reply_stream(inputs)`：流式产出 `AgentEvent`
- `reply(inputs)`：消费同一条 `_reply()` 事件流，最终返回 `Msg`

Web channel 使用 `reply_stream()`，由 `ChatService` 将事件写入 MessageBus；同步调用场景可以使用 `reply()`。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:191` — `reply_stream()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:215` — `reply()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py:366` — `ChatService` 消费 `agent.reply_stream()`

### _reply_impl 主循环

`_reply_impl()` 的核心流程：

1. 区分输入是新消息，还是 `UserConfirmResultEvent` / `ExternalExecutionResultEvent`
2. `_check_incoming_event()` 判断当前是否处于等待确认或外部执行结果状态
3. 若是恢复事件，调用 `_handle_incoming_event()` 更新 tool call state/context
4. 若是新消息，调用 `_handle_incoming_messages()` 写入 context，生成新的 `reply_id`，重置 `cur_iter`
5. 产出 `ReplyStartEvent`
6. 在 `cur_iter < max_iters` 内循环：
   - `_check_next_action()` 判断应退出、reasoning 还是 acting
   - reasoning 前调用 `compress_context()`
   - `_reasoning()` 调模型并把流式 chunk 转成事件
   - `_batch_tool_calls()` 根据工具属性拆成 sequential/concurrent batch
   - 执行 tool calls，遇到用户确认或外部执行需求则暂停并返回等待消息
7. 超过 max_iters 时产出 `ExceedMaxItersEvent`、`ReplyEndEvent` 和兜底 `AssistantMsg`

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:533` — `_reply_impl()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:569` — `_check_incoming_event()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:582` — 新 reply 重置 `reply_id` / `cur_iter`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:598` — reasoning-acting loop
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:676` — max_iters 兜底事件

### Reasoning 是事件转换器

`_reasoning_impl()` 做三件事：

1. `_prepare_model_input()` 组装 system prompt、summary、context、tool schemas
2. `_call_model()` 执行模型调用，支持重试、fallback model 和 `on_model_call` middleware
3. 将 `ChatResponse` 或 streaming chunks 转为 `ModelCallStart/End`、Text/Thinking/Tool/Data block events，并把 completed response 保存进 context

若模型最终没有生成 `ToolCallBlock`，`_reasoning_impl()` 直接产出最终 `AssistantMsg`，主循环结束。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:707` — `_reasoning_impl()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:2236` — `_prepare_model_input()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:2265` — `_call_model()` 重试/fallback/middleware
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:779` — `ModelCallEndEvent`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:789` — 保存 completed response 到 context

### Acting 拆成 permission / execution / context 三层

`_execute_tool_call()` 包办 tool lifecycle：

- 校验工具是否可用
- 用 tool schema 解析并校验输入
- 调 `PermissionEngine.check_permission()`
- 需要用户确认时产出 `RequireUserConfirmEvent`
- 外部工具产出 `RequireExternalExecutionEvent`
- 允许执行时产出 `ToolResultStartEvent`
- 调 `_acting()` 执行工具
- 将 tool chunk 转成流式事件
- 对最终 `ToolResponse` 做 tool-result compression/offload
- 写 context，并把 tool call state 改为 `FINISHED`

`_acting()` 本身只包裹 `toolkit.call_tool()`，是 `on_acting` middleware 的 hook point。permission checking、input validation、context writes 都在 `_execute_tool_call()` 外层，避免 middleware 背景化执行时直接写 agent context。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1311` — `_execute_tool_call()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1437` — permission ASK/PASSTHROUGH
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1487` — external tool 暂停
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1502` — `_acting()` hook point
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1570` — tool result 写 context 并结束
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1612` — `_acting()` middleware wrapper

### Tool Call Batch 由工具属性决定

`_batch_tool_calls()` 会读取 tool metadata：

- 找不到 tool 或 tool 标记 `is_concurrency_safe=True` → 放入 concurrent batch
- 否则放入 sequential batch

concurrent 执行使用 `asyncio.gather(return_exceptions=True)`，同时通过 shared queue 把各 tool 的事件流持续吐出；gather 完成后再统一收集异常并抛 `ExceptionGroup`。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1034` — `_batch_tool_calls()`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1078` — sequential execution
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1130` — concurrent execution
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py:1219` — `asyncio.gather(..., return_exceptions=True)`

### Middleware 是当前扩展机制

`MiddlewareBase` 支持：

- `on_reply`
- `on_reasoning`
- `on_acting`
- `on_model_call`
- `on_compress_context`
- `on_system_prompt`

前五个是 onion pattern：middleware 调 `next_handler()` 包裹后续链路；`on_system_prompt` 是 transformer pattern，按顺序修改 prompt string。

源码依据：

- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/middleware/_base.py:13` — middleware hook 总览
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/middleware/_base.py:65` — `on_reply`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/middleware/_base.py:92` — `on_reasoning`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/middleware/_base.py:114` — `on_acting`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/middleware/_base.py:160` — `on_model_call`
- `/Users/neo/Desktop/project/git/agentscope/src/agentscope/middleware/_base.py:208` — `on_system_prompt`

---

## 设计亮点

### 1. Event-first query loop

Agent loop 不只返回最终文本，而是把 model call、text/thinking/tool/data block、tool result、confirmation、external execution、reply start/end 都转成结构化事件。Web/SSE、日志、trace、UI 可以共享同一份事件语义。

### 2. 外部确认/外部执行是 loop 原生状态

`UserConfirmResultEvent` 和 `ExternalExecutionResultEvent` 不是旁路 API，而是 `reply_stream(inputs)` 的合法输入类型。agent 可在工具确认或外部执行时暂停，下一次 run 用事件恢复。

### 3. 并发工具调用受 tool metadata 控制

并发不是单纯“LLM 一次生成多个 tool 就全部 gather”。AgentScope 会按 `is_concurrency_safe` 分 batch，把有副作用或不可并发的工具串行化。这比完全依赖 LLM 自觉更稳。

### 4. Context / tool result 压缩在 loop 内部

reasoning 前调用 `compress_context()`，tool result 写入前调用 `_split_tool_result_for_compression()`。大输出会被截断，并可通过 offloader 保存剩余内容，避免 tool result 直接冲爆上下文。

### 5. Middleware 替代旧元类 hook

当前扩展点是显式 middleware 列表，而不是类级全局 hook/metaclass 注入。这样更容易按 session/user 注入审计、tool offload、inbox、trace、system prompt 修改等能力。

---

## 局限性

### 1. 终止语义仍主要依赖“无 tool call”

主循环在模型没有生成 tool call 时结束；是否真的完成任务仍由模型输出质量决定。`max_iters` 是硬上限，不是语义完成判定。

### 2. concurrent batch 不表达工具依赖图

`is_concurrency_safe` 只说明工具是否安全并发，不说明工具调用之间的数据依赖。如果模型一次生成依赖链工具调用，框架无法自动推导 DAG，只能靠工具 metadata 和 prompt 约束。

### 3. 外部确认会暂停整个 run

遇到 `RequireUserConfirmEvent` 或 `RequireExternalExecutionEvent` 后，当前 `_reply_impl()` 返回等待消息。这个设计利于恢复，但也意味着一个需要确认的工具会阻塞后续 batch。

### 4. state-injected 工具的后台 offload 仍有风险

源码注释指出，`is_state_injected=True` 的工具会拿到 live `agent.state`；若被 `on_acting` middleware offload 到后台任务，可能产生并发状态修改风险。agent-os 如果实现 tool offload，需要明确禁止或序列化 state-injected tools。

---

## 对 agent-os 的借鉴

P0：query loop 应先产出结构化事件，再由 channel 决定如何显示。不要把 CLI 文本流、Web SSE、trace 三套输出做成三套逻辑。

P0：确认/外部执行/后台任务完成都应是 loop 的可恢复输入事件，而不是 handler 里直接改数据库。这样本地 agent 和 Web agent 可以共享恢复机制。

P1：tool metadata 至少包含 `is_concurrency_safe`、`is_external_tool`、`is_state_injected` 三类决策信息。它们直接影响 batch、permission、offload 和恢复策略。

P1：middleware 比继承 hook 更适合产品化 agent-os。local shell 可以注入轻量 tracing；Web shell 可以注入 inbox、state-change publish、tool offload、tenant audit。

P1：context compression 和 tool-result offload 应进入 loop 的固定生命周期点，而不是作为“出问题后补救”的外部清理任务。

---

## 来源

- 源码版本：`v2.0.1-11-g0e5418e8`
- 分析深度：源码级
- 核心文件：
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/agent/_agent.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/state/_state.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/middleware/_base.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/_service/_chat.py`
  - `/Users/neo/Desktop/project/git/agentscope/src/agentscope/app/middleware/`
