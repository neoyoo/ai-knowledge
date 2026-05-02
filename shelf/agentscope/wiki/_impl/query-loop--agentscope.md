---
title: "query-loop——agentscope"
category: L2
parent: "[[query-loop]]"
source: agentscope
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "query-loop"
created: "2026-04-15"
confidence: high
---

## 概述

AgentScope 用 `ReActAgent` 实现经典 ReAct 主循环：在 `reply()` 中以 `for _ in range(max_iters)` 驱动 `_reasoning → _acting` 的迭代，退出条件是 LLM 不再输出 tool_use block。整个调用链完全异步（asyncio），工具调用支持顺序或并发两种执行模式，且每个环节（reply / reasoning / acting / observe / print）都被 metaclass 自动注入双向 hook，实现零侵入的全链路拦截。

---

## 架构分析

### 层次结构

```
AgentBase (StateModule, metaclass=_AgentMeta)
  └─ ReActAgentBase (metaclass=_ReActAgentMeta)
       └─ ReActAgent  ← 唯一内置的全功能 agent 实现
```

- `AgentBase`：定义 `reply / observe / print / __call__` 接口，通过 `_AgentMeta` 元类在类创建时将这三个方法替换为带 hook 包装的版本。
- `ReActAgentBase`：在 `AgentBase` 基础上增加 `_reasoning / _acting` 抽象方法，由 `_ReActAgentMeta` 再次注入 hook。
- `ReActAgent`：唯一的产品级实现，持有 model / formatter / toolkit / memory / long_term_memory / knowledge 等所有依赖。

### `__call__` 是入口，不是 `reply`

用户调用 `await agent(msg)` 触发的是 `AgentBase.__call__`：

```python
async def __call__(self, *args, **kwargs) -> Msg:
    self._reply_id = shortuuid.uuid()
    self._reply_task = asyncio.current_task()
    try:
        reply_msg = await self.reply(*args, **kwargs)
    except asyncio.CancelledError:
        reply_msg = await self.handle_interrupt(*args, **kwargs)
    finally:
        if reply_msg:
            await self._broadcast_to_subscribers(reply_msg)
        self._reply_task = None
    return reply_msg
```

`__call__` 负责两件事：捕获 `CancelledError` 实现中断（realtime steering），以及回复完成后向所有 MsgHub 订阅者广播（自动剥除 thinking block）。

### 核心循环结构（reply）

`ReActAgent.reply()` 完整流程：

```
1. memory.add(msg)                          ← 输入入库
2. _retrieve_from_long_term_memory(msg)     ← 长期记忆检索（可选）
3. _retrieve_from_knowledge(msg)            ← RAG 检索（可选）
4. 注册 / 注销 generate_response tool       ← 结构化输出管理
5. for _ in range(max_iters):
   a. _compress_memory_if_needed()          ← token 超阈压缩（可选）
   b. msg_reasoning = await _reasoning(tool_choice)
   c. futures = [_acting(tc) for tc in tool_use_blocks]
   d. 并发或顺序执行 futures
   e. 判断退出：无 tool_use block → break
6. 若超迭代未退出 → _summarizing()
7. 静态长期记忆写回（可选）
8. return reply_msg
```

### Pipeline 层（agent 间编排）

Pipeline 不是 query-loop 本身，而是 agent 间的调用编排：

| 类型 | 实现 | 特点 |
|------|------|------|
| `sequential_pipeline` | 函数式，`for agent in agents: msg = await agent(msg)` | 链式传递，每步等待上一步 |
| `fanout_pipeline` | `asyncio.gather` 或顺序执行 | 广播同一输入到多 agent，并发收集结果 |
| `SequentialPipeline` / `FanoutPipeline` | 上述函数的类包装 | 可复用实例 |
| `stream_printing_messages` | asyncio.Queue 生成器 | 将 agent 内部 `print()` 调用的中间消息暴露为异步生成器，用于 SSE 场景 |

### MsgHub（多 agent 对话广播）

`MsgHub` 是一个 async context manager，在 `__aenter__` 时将参与 agent 互相设置为订阅者，`__aexit__` 时清除。当任一 agent 完成 `reply` 后，`__call__` 中的 `_broadcast_to_subscribers` 自动将其输出 `observe` 注入所有其他 agent 的 memory，无需手动传递消息。

### Hook 系统（元类注入）

`_AgentMeta.__new__` 在类定义时用 `_wrap_with_hooks` 替换 `reply / print / observe`：

```python
# _agent_meta.py
class _AgentMeta(type):
    def __new__(mcs, name, bases, attrs):
        for func_name in ["reply", "print", "observe"]:
            if func_name in attrs:
                attrs[func_name] = _wrap_with_hooks(attrs[func_name])
        return super().__new__(mcs, name, bases, attrs)
```

`_wrap_with_hooks` 执行顺序：`instance_pre_hooks → class_pre_hooks → original_func → instance_post_hooks → class_post_hooks`，每个 hook 都可以修改入参（pre）或返回值（post）。

---

## 关键代码路径

### 路径 1：完整 ReAct 主循环

```
agent(msg)                           # AgentBase.__call__
  └─ await self.reply(msg)           # _wrap_with_hooks 包装后的 reply
       └─ ReActAgent.reply(msg)
            ├─ await memory.add(msg)
            ├─ await _retrieve_from_long_term_memory(msg)
            ├─ await _retrieve_from_knowledge(msg)
            └─ for _ in range(max_iters):
                 ├─ await _compress_memory_if_needed()
                 ├─ msg_r = await self._reasoning(tool_choice)
                 │    ├─ prompt = await formatter.format([sys, *memory.get_memory()])
                 │    ├─ res = await self.model(prompt, tools=toolkit.get_json_schemas(), tool_choice=tool_choice)
                 │    ├─ async for chunk in res:   # streaming
                 │    │    └─ await self.print(msg, last=False)
                 │    ├─ await self.print(msg, last=True)
                 │    └─ await memory.add(msg)
                 ├─ futures = [self._acting(tc) for tc in msg_r.get_content_blocks("tool_use")]
                 ├─ structured_outputs = await asyncio.gather(*futures)  # parallel
                 │    or [await f for f in futures]                       # sequential
                 └─ if not msg_r.has_content_blocks("tool_use"): break   # exit condition
```

### 路径 2：_acting 工具执行

```python
# agent/_react_agent.py
async def _acting(self, tool_call: ToolUseBlock) -> dict | None:
    tool_res = await self.toolkit.call_tool_function(tool_call)
    async for chunk in tool_res:
        tool_res_msg.content[0]["output"] = chunk.content
        await self.print(tool_res_msg, chunk.is_last)
        if chunk.is_interrupted:
            raise asyncio.CancelledError()
    await self.memory.add(tool_res_msg)
    # 若是 generate_response 且验证通过，返回 structured_output
    if tool_call["name"] == self.finish_function_name and chunk.metadata["success"]:
        return chunk.metadata.get("structured_output")
    return None
```

### 路径 3：中断处理（Realtime Steering）

```python
# AgentBase.__call__
try:
    reply_msg = await self.reply(*args, **kwargs)
except asyncio.CancelledError:
    reply_msg = await self.handle_interrupt(*args, **kwargs)

# 外部触发中断
async def interrupt(self, msg=None) -> None:
    if self._reply_task and not self._reply_task.done():
        self._reply_task.cancel(msg)
```

### 路径 4：记忆压缩触发

```python
# agent/_react_agent.py
async def _compress_memory_if_needed(self):
    to_compressed = await self.memory.get_memory(exclude_mark=_MemoryMark.COMPRESSED)
    prompt = await formatter.format([sys, *to_compressed])
    n_tokens = await compression_config.agent_token_counter.count(prompt)
    if n_tokens > compression_config.trigger_threshold:
        res = await compression_model(compression_prompt, structured_model=SummarySchema)
        await memory.update_compressed_summary(summary_template.format(**res.metadata))
        await memory.update_messages_mark(msg_ids, new_mark=_MemoryMark.COMPRESSED)
```

### 路径 5：Stream 消息暴露（pipeline）

```python
# pipeline/_functional.py
async def stream_printing_messages(agents, coroutine_task, queue=None, ...):
    queue = queue or asyncio.Queue()
    for agent in agents:
        agent.set_msg_queue_enabled(True, queue)
    task = asyncio.create_task(coroutine_task)
    task.add_done_callback(lambda _: queue.put_nowait(end_signal))
    while True:
        printing_msg = await queue.get()
        if printing_msg == end_signal:
            break
        yield msg, last
```

---

## 设计亮点

### 1. 元类注入 Hook，零侵入全链路拦截

用 `_AgentMeta` / `_ReActAgentMeta` 在类定义时（非实例化时）替换目标方法，开发者继承后无需任何额外调用，pre/post hook 自动生效。Hook 同时支持类级（所有实例共享）和实例级两种粒度，且执行顺序确定（instance hooks 优先于 class hooks）。

### 2. `__call__` 与 `reply` 分离——中断点设计

将"是否 cancelled"和"广播订阅者"逻辑放在 `__call__` 而非 `reply`，使 `reply` 只关注生成逻辑。外部任何时候调用 `agent.interrupt()` 取消 asyncio task，`CancelledError` 被 `__call__` 捕获后调用 `handle_interrupt`，原始请求参数仍然可访问，支持定制化中断响应（如生成"我注意到你打断了我"这类消息）。

### 3. thinking block 的自动剥除

`_broadcast_to_subscribers` 在广播前调用 `_strip_thinking_blocks`，确保 chain-of-thought 内容不会泄露给其他 agent，保持推理的私密性，不需要开发者手动过滤。

### 4. 并发工具调用（parallel_tool_calls）

当 LLM 在一次 reasoning 中输出多个 tool_use block 时，`parallel_tool_calls=True` 会用 `asyncio.gather` 并发执行所有工具，而非逐个等待。对于独立工具（如多次文件读取）可显著降低延迟。

### 5. 结构化输出通过 tool_use 实现（generate_response trick）

需要结构化输出时，AgentScope 不依赖模型的原生 structured output API（各家不统一），而是动态注册一个 `generate_response` 工具函数，用 Pydantic 模型描述 schema，强制 `tool_choice="required"` 让 LLM 调用该工具，工具执行时做 `model_validate` 校验，失败时写回错误信息继续迭代。这种方式在所有支持 function calling 的模型上均可工作。

### 6. 记忆压缩内置在循环里

`_compress_memory_if_needed` 在每次 reasoning 之前调用，而非在 reply 结束后。这意味着超长对话中途就会触发压缩，不需要等到 context 溢出。压缩结果用 `_MemoryMark.COMPRESSED` 标记，原始消息保留但在 `get_memory` 时被过滤掉，换成 summary 替代，兼顾可追溯性与窗口控制。

### 7. stream_printing_messages 解耦流式输出

通过 asyncio.Queue 将 agent 内部的每次 `print()` 调用转换为外部可消费的异步生成器，不需要修改 agent 逻辑即可实现 SSE/WebSocket 推送。调用方只需 `async for msg, last in stream_printing_messages(agents, task)` 即可获取实时中间消息。

---

## 局限性

### 1. 循环终止条件过于简单

退出条件是"没有 tool_use block"，即 LLM 输出纯文本时退出。这在理论上正确，但没有语义检查（例如 LLM 是否真的完成了任务还是只是卡住了）。`max_iters` 是唯一的安全阀，超出后调用 `_summarizing()` 生成一个强制性总结，但这个总结质量无法保证。

### 2. 结构化输出依赖 function calling，不支持纯文本模型

`generate_response` trick 要求模型支持 tool_use / function calling。对于不支持工具的模型（如某些开源模型），结构化输出路径完全不可用。

### 3. 记忆压缩在循环内部会增加额外的 LLM 调用延迟

`_compress_memory_if_needed` 在每次 reasoning 前都检查一次，一旦触发会增加一次完整的 LLM 调用。在高频低延迟场景（如对话系统）中，这可能造成明显的卡顿，且目前没有异步后台压缩的设计。

### 4. parallel_tool_calls 缺乏依赖感知

`asyncio.gather` 并发执行所有工具，但工具之间如果有依赖关系（如 tool B 需要 tool A 的输出），框架本身无法检测，只能依赖 LLM 在生成工具调用时自行保证顺序。

### 5. MsgHub 的广播是全量 observe，无过滤机制

MsgHub 中每个 agent 的 reply 都会 broadcast 给所有其他参与者，缺乏内容过滤或路由机制。在大型多 agent 场景中，无关信息会累积进所有 agent 的 memory，增加无效 context。

### 6. 全局类级 hook 共享状态，测试隔离困难

`_class_pre_reply_hooks` 等字典是类变量，所有实例共享。在测试或多租户场景中注册的 class hook 会影响所有 instance，需要手动调用 `clear_class_hooks()` 清理，容易遗漏。

---

## 来源

- 源码版本：0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12
- 分析深度：源码级
- 核心文件：
  - `src/agentscope/agent/_agent_base.py` — AgentBase、__call__、hook 注册、broadcast
  - `src/agentscope/agent/_agent_meta.py` — _AgentMeta、_ReActAgentMeta、_wrap_with_hooks
  - `src/agentscope/agent/_react_agent_base.py` — ReActAgentBase、reasoning/acting 抽象接口
  - `src/agentscope/agent/_react_agent.py` — ReActAgent、reply 主循环、_reasoning、_acting、压缩、RAG
  - `src/agentscope/pipeline/_functional.py` — sequential_pipeline、fanout_pipeline、stream_printing_messages
  - `src/agentscope/pipeline/_class.py` — SequentialPipeline、FanoutPipeline
  - `src/agentscope/pipeline/_msghub.py` — MsgHub、auto-broadcast
  - `src/agentscope/model/_model_base.py` — ChatModelBase、tool_choice 验证
