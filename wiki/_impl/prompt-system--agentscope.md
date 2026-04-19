---
title: "prompt-system——agentscope"
category: L2
parent: "[[prompt-system]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "prompt-system"
created: "2026-04-15"
confidence: high
---

## 概述

AgentScope 将 prompt 组装完全外置为独立的 `formatter` 层：`Msg` 对象列表 → `FormatterBase.format()` → provider-specific `list[dict]`。每个 provider 有对应的 `ChatFormatter`（单 agent）和 `MultiAgentFormatter`（多 agent）两个变体，所有实现均继承自 `TruncatedFormatterBase`，支持令牌计数 + 自动截断。formatter 由外部注入 agent，不与 model 耦合，agent 调用 `formatter.format(msgs)` 得到最终 prompt，再传给 model。

---

## 架构分析

### 整体分层

```
Agent (ReactAgent / ...)
  └─ formatter: FormatterBase          # 注入依赖，不耦合 model
       ├─ format(list[Msg]) → list[dict]   # 入口，@trace_format 装饰
       ├─ _format(list[Msg]) → list[dict]  # 每个 provider 实现
       ├─ _truncate(list[Msg]) → list[Msg] # 超 token 限时削减历史
       └─ _count(list[dict]) → int | None  # 委托 TokenCounterBase
```

`FormatterBase` 是抽象根，`TruncatedFormatterBase` 在其上加了截断循环，所有真实 formatter 都继承后者。A2A 协议的 `A2AChatFormatter` 例外，它直接继承 `FormatterBase` 且返回值是 `a2a.types.Message` 而非 `list[dict]`。

### 双模式分工：Chat vs MultiAgent

| 维度 | ChatFormatter | MultiAgentFormatter |
|------|--------------|---------------------|
| 定位 | 1 user + 1 agent 对话 | N 个 agent 参与的多方对话 |
| 历史表示 | 每条消息保持独立角色 | 非工具消息合并入 `<history>…</history>` 用户消息 |
| 工具消息 | 直接逐条输出 | 工具序列 → 代理给 ChatFormatter 处理 |
| 多模态 | 各自格式化 | 多模态块穿插在历史文本中 |

MultiAgentFormatter 的核心策略：把历史轮次的多个 agent 发言压成一条 user 消息（文本拼接为 `name: text\n`），只保留最近的工具调用序列作为独立消息。这样解决了 OpenAI/Anthropic 等不支持 `assistant→assistant` 连续消息的限制。

### Msg 内容块模型

`Msg.content` 是 typed dict 的列表，支持以下 block 类型：

- `TextBlock` — `{"type": "text", "text": str}`
- `ImageBlock` — `{"type": "image", "source": URLSource | Base64Source}`
- `AudioBlock` / `VideoBlock` — 同构
- `ToolUseBlock` — `{"type": "tool_use", "id", "name", "input"}`
- `ToolResultBlock` — `{"type": "tool_result", "id", "name", "output"}`
- `ThinkingBlock` — Anthropic 扩展思考块，`{"type": "thinking", ...}`

formatter 按 block 类型逐一分发到 provider 特定格式；未知 block 类型只打 warning 不抛错，保证容错。

### 截断机制

`TruncatedFormatterBase.format()` 用 while 循环实现渐进截断：

```
while True:
    formatted = await self._format(msgs)
    n_tokens = await self._count(formatted)
    if n_tokens is None or max_tokens is None or n_tokens <= max_tokens:
        return formatted
    msgs = await self._truncate(msgs)   # 从最老非系统消息开始删
```

`_truncate()` 有一个关键约束：tool_use 和 tool_result 必须成对删除，否则 provider API 会报错。它用 `tool_call_ids` 集合追踪配对关系，找到第一个"配对清空"边界才停止。

### Provider 差异处理

| Provider | 主要差异点 |
|----------|-----------|
| OpenAI | 工具消息 role=`"tool"`，audio block 过滤掉 assistant 端的输出 audio |
| Anthropic | tool_result 必须包在 role=`"user"` 消息里；system 只允许首条；保留 thinking block |
| DashScope | content 不能为 None（用 `[]` 代替）；纯文本内容可降级为 string（HF tokenizer 兼容） |
| Gemini | role 只有 `"user"/"model"`；tool_result 用 `function_response` 包裹；media 必须 inline_data |
| Ollama | 结构与 OpenAI 相似但 audio 不支持 |
| DeepSeek | 不支持 vision（`support_vision=False`），无多模态 block |

### 图像提升（promote_tool_result_images）

所有支持 vision 的 formatter 都有 `promote_tool_result_images` 参数（DashScope 还有 audio/video 变体）。当工具返回图片时，大多数 API 不允许在 tool_result 里放图，该参数开启后会将图片提取出来，插入一条新 user 消息并附上系统说明文字 `<system-info>…</system-info>`。实现上直接往 `msgs` 列表中插入新 `Msg`，循环指针 `i` 不变，下一轮自然会处理插入的消息。

### OpenTelemetry 追踪

`TruncatedFormatterBase.format()` 上有 `@trace_format` 装饰器（定义于 `tracing/_trace.py`）。追踪属性包括 formatter 类名、输入 msg 数量、输出 dict 数量，以 OpenTelemetry span 形式上报，对 tracing 未启用时零开销直接透传。

### A2A 协议支持

`A2AChatFormatter` 是特殊的双向 formatter：
- `format(list[Msg]) → a2a.types.Message`：AgentScope → A2A
- `format_a2a_message(name, Message) → Msg`：A2A → AgentScope
- `format_a2a_task(name, Task) → list[Msg]`：A2A Task（含 artifacts）→ AgentScope

这让 AgentScope agent 能无缝接入 Google A2A 协议网络，充当 A2A client 或 server。

---

## 关键代码路径

### 主调用链：ReAct Agent → formatter → model

```python
# agentscope/agent/_react_agent.py, _reasoning()
prompt = await self.formatter.format(
    msgs=[
        Msg("system", self.sys_prompt, "system"),
        *await self.memory.get_memory(exclude_mark=_MemoryMark.COMPRESSED),
    ],
)
res = await self.model(prompt, tools=self.toolkit.get_json_schemas(), ...)
```

formatter 完全独立于 model，由 `ReactAgent.__init__` 注入：

```python
# agentscope/agent/_react_agent.py
def __init__(self, ..., formatter: FormatterBase, ...):
    self.formatter = formatter
```

### TruncatedFormatterBase.format() — 截断主循环

```python
# agentscope/formatter/_truncated_formatter_base.py
@trace_format
async def format(self, msgs: list[Msg], **kwargs) -> list[dict[str, Any]]:
    self.assert_list_of_msgs(msgs)
    msgs = deepcopy(msgs)          # 不污染原始消息列表
    while True:
        formatted_msgs = await self._format(msgs)
        n_tokens = await self._count(formatted_msgs)
        if n_tokens is None or self.max_tokens is None or n_tokens <= self.max_tokens:
            return formatted_msgs
        msgs = await self._truncate(msgs)
```

### _group_messages() — 多 agent 消息分组

```python
# agentscope/formatter/_truncated_formatter_base.py
@staticmethod
async def _group_messages(msgs: list[Msg]) -> AsyncGenerator[
    Tuple[Literal["tool_sequence", "agent_message"], list[Msg]], None
]:
    # 按 tool_use/tool_result block 的存在与否分组
    # 连续同类消息合并为一组，切换时 yield
```

输出两类 group：`"tool_sequence"` 交给 `_format_tool_sequence()`，`"agent_message"` 交给 `_format_agent_message()`。

### OpenAI MultiAgent — 历史压缩

```python
# agentscope/formatter/_openai_formatter.py, OpenAIMultiAgentFormatter._format_agent_message()
for msg in msgs:
    for block in msg.get_content_blocks():
        if block["type"] == "text":
            accumulated_text.append(f"{msg.name}: {block['text']}")
        elif block["type"] == "image":
            images.append(_format_openai_image_block(block))

# 包裹为单条 user 消息
conversation_blocks[0]["text"] = (
    conversation_history_prompt + "<history>\n" + conversation_blocks[0]["text"]
)
# ...
user_message = {"role": "user", "content": content_list}
```

### _truncate() — 配对删除保障

```python
# agentscope/formatter/_truncated_formatter_base.py
tool_call_ids = set()
for i in range(start_index, len(msgs)):
    msg = msgs[i]
    for block in msg.get_content_blocks("tool_use"):
        tool_call_ids.add(block["id"])
    for block in msg.get_content_blocks("tool_result"):
        tool_call_ids.remove(block["id"])   # 配对消除
    if len(tool_call_ids) == 0:
        return msgs[:start_index] + msgs[i + 1:]   # 找到完整边界，截掉前面
```

### DashScope 的 content 降级

```python
# agentscope/formatter/_dashscope_formatter.py, _reformat_messages()
# 全是文本时，把 list[{"text": ...}] 降级为单个 string
# 原因：HuggingFace tokenizer 不支持 list 格式的 content
for message in messages:
    if is_all_text:
        message["content"] = "\n".join(texts)
```

---

## 设计亮点

### 1. Formatter 完全独立于 Model

formatter 与 model 完全解耦，通过构造函数注入 agent。这意味着同一个 model（如 OpenAI 兼容 API）可以配不同 formatter（单 agent vs 多 agent），也可以在 compression 路径中换用另一个 formatter，而无需修改 model 代码。

### 2. 双模式 formatter 解决多 agent 兼容问题

绝大多数 LLM API 只支持 user/assistant 严格交替，多 agent 场景天然产生多个 assistant 连续发言的问题。AgentScope 通过 MultiAgentFormatter 把历史压成带 `<history>` 标签的单条 user 消息，同时保留工具调用序列的独立结构，是一个务实的工程解法。

### 3. 截断循环 + 工具配对删除

截断时不能随意删中间消息，必须以完整的 tool_use+tool_result 对为最小删除单元。AgentScope 的 `_truncate()` 用 set 追踪未匹配的 tool_call_id，找到第一个"队列清空"边界才截断，精确处理了这个约束。

### 4. promote_tool_result_images 的优雅实现

直接往 `msgs` list 插入新消息，while 循环的 `i` 不变、下一轮处理插入消息，既复用了已有的格式化路径，又避免额外递归，代码简洁。

### 5. A2A 双向桥接

`A2AChatFormatter` 同时实现 AgentScope → A2A 和 A2A → AgentScope 两个方向的转换，且处理了 `Task`（含 artifacts）这种复杂结构，使得 AgentScope agent 可以零改动地接入 A2A 协议网络。

### 6. OpenTelemetry 追踪内建

`@trace_format` 装饰器在 formatter 公共入口点注入追踪，无需每个 provider 子类单独实现，同时在 tracing 未启用时零开销。formatter span 包含输入输出的消息数量，便于调试截断行为。

---

## 局限性

### 1. MultiAgent 历史压缩损失结构信息

把多条 agent 消息压成一段文本（`name: text\n` 拼接），虽然解决了 API 兼容问题，但消息边界信息丢失，图片和文字之间的顺序关系也可能被打乱。对于强依赖多 agent 角色区分的场景，这种压缩会降低模型的理解质量。

### 2. 截断策略固定，不可配置

`_truncate()` 只实现了"从最老消息开始删整对"的简单策略，没有提供替换入口（只在注释中提示"可 override"）。无法配置按重要性保留、按 agent 身份保留、或基于语义的截断策略。

### 3. AnthropicChatFormatter 不支持多 agent

`AnthropicChatFormatter.support_multiagent = False`，如果用 Anthropic 做多 agent 只能用 `AnthropicMultiAgentFormatter`，但后者会触发历史压缩，不能保留完整对话结构。

### 4. DashScope 的 `_reformat_messages` 是补丁性质

专门为 HuggingFace tokenizer 做 content list → string 降级，是外部依赖限制倒逼出来的补丁，增加了格式转换的隐式副作用，调试时容易困惑。

### 5. 工具调用中多模态返回有局限

`convert_tool_result_to_string()` 会把 base64 图片存为本地文件，把 URL 图片的路径插入文本，这一操作有副作用（磁盘 I/O），且本地文件路径对分布式场景不可移植。

### 6. formatter 没有校验 capability

formatter 上声明了 `support_tools_api`、`support_vision`、`support_multiagent`，但这些字段只是标记，agent 和用户需要自己读取并判断，框架不会在传入不支持的 block 时自动降级或报错（只打 warning）。

---

## 关系

- [[prompt-system]] — 本页是其 AgentScope 实现详情
- [[tool-system--agentscope]] — tool_use/tool_result block 格式由 formatter 负责转换
- [[context-memory--agentscope]] — memory.get_memory() 的输出直接传入 formatter.format()
- [[multi-agent--agentscope]] — MultiAgentFormatter 是多 agent 场景的核心适配层

---

## 来源

- 源码版本：`0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12`
- 分析深度：源码级
- 主要文件：
  - `agentscope/formatter/_formatter_base.py`
  - `agentscope/formatter/_truncated_formatter_base.py`
  - `agentscope/formatter/_openai_formatter.py`
  - `agentscope/formatter/_anthropic_formatter.py`
  - `agentscope/formatter/_dashscope_formatter.py`
  - `agentscope/formatter/_gemini_formatter.py`
  - `agentscope/formatter/_a2a_formatter.py`
  - `agentscope/agent/_react_agent.py`（formatter 使用侧）
  - `agentscope/tracing/_trace.py`（@trace_format 实现）
