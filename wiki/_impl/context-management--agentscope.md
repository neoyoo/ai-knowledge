---
title: "context-management——agentscope"
category: L2
parent: "[[context-management]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "context-management"
created: "2026-04-15"
confidence: high
---

## 概述

AgentScope 用三层独立机制管理 context window：**Token 计数层**（5 种后端的统一 async 接口）、**Formatter 截断层**（format 时按 token 限制自动 drop 旧消息）、**Memory 压缩层**（agent 级 LLM 驱动压缩，把超出阈值的历史摘要化后继续保留 summary）。三层正交、可按需组合。

---

## 架构分析

### 整体数据流

```
MemoryBase (存储所有历史 Msg)
    ↓ get_memory()
TruncatedFormatterBase._format(msgs)   ← 格式化为 API 所需 dict
    ↓ _count(formatted_msgs)           ← 调用 TokenCounterBase.count()
    ↓ 超限？_truncate(msgs)            ← drop 最旧的 turn（保持 tool_call/result 配对）
    ↓ 再 _format + _count …直到合法
最终 prompt → 模型 API

ReActAgent.reply()
    ↓ 每轮 loop 在 _reasoning() 前调用 _compress_memory_if_needed()
       token_counter.count(formatter.format(sys+history))
       超过 trigger_threshold？
           LLM 生成 SummarySchema (结构化 JSON)
           update_compressed_summary(summary_text)   → 写入 memory._compressed_summary
           update_messages_mark(ids, COMPRESSED)     → 标记已压缩 msgs
    ↓ get_memory(prepend_summary=True) → [summary_msg] + 未压缩 msgs
```

### 1. Token 计数层（`agentscope/token/`）

五种实现，统一基类 `TokenCounterBase`：

| 实现类 | 后端 | 特点 |
|---|---|---|
| `OpenAITokenCounter` | tiktoken（本地） | 按 OpenAI 官方算法：文本/图片/tool_calls 分路计算；图片按模型系列（gpt-4o/gpt-4.1/o1 等）差异化 tile 算法 |
| `AnthropicTokenCounter` | Anthropic 计数 API（网络调用） | 调用 `client.messages.count_tokens()`，原生精确，需要 API key |
| `GeminiTokenCounter` | Google genai SDK（网络调用） | `client.models.count_tokens()`，支持 tools 配置透传 |
| `HuggingFaceTokenCounter` | transformers AutoTokenizer（本地） | `apply_chat_template(tokenize=True)`，需要 chat_template，支持任意 HF 模型 |
| `CharTokenCounter` | 字符数（本地，降级方案） | `str(msg)` 字符数；不适合多模态（base64 会膨胀） |

所有 `count()` 均为 `async`，接受 `messages: list[dict]` 和可选 `tools: list[dict]`。

OpenAI 图片计数细节（`_openai_token_counter.py`）：
- `gpt-4o/gpt-4.1/gpt-4.5`：base=85, tile=170；先缩放至 2048×2048 盒，再缩最短边至 768，按 512px tiles 计算
- `gpt-4.1-mini/nano/o4-mini`：另有 patch 算法（`patches = min(ceil(w/32)*ceil(h/32), 1536)`）
- tools 计数参考 OpenAI cookbook，有 `func_init / prop_init / enum_item` 等经验值

### 2. Formatter 截断层（`agentscope/formatter/_truncated_formatter_base.py`）

`TruncatedFormatterBase` 持有 `token_counter` 和 `max_tokens`，在 `format()` 入口形成截断循环：

```python
# TruncatedFormatterBase.format()
while True:
    formatted_msgs = await self._format(msgs)       # 子类实现（OpenAI/Anthropic/Gemini 等格式）
    n_tokens = await self._count(formatted_msgs)    # 调用 token_counter.count()
    if n_tokens is None or max_tokens is None or n_tokens <= max_tokens:
        return formatted_msgs
    msgs = await self._truncate(msgs)               # 截断最旧 turn
```

`_truncate()` 策略：
- 系统消息永远保留（若系统消息本身超限则 raise ValueError）
- 从最旧消息（start_index=1）开始，按 turn 粒度向后扫描
- 使用 `tool_call_ids` 集合追踪配对：`tool_use` 和对应 `tool_result` 必须整批删除，不能割裂
- 找到第一个"配对完整"的位置后，删除 `msgs[start_index:i+1]`，返回剩余 msgs

`_format()` 内部通过 `_group_messages()` 将消息流拆成 `agent_message` 和 `tool_sequence` 两类组，分别委托不同子方法处理。

`OpenAIChatFormatter` 和 `OpenAIMultiAgentFormatter` 均继承 `TruncatedFormatterBase`，构造时接受 `token_counter` 和 `max_tokens` 参数，可选启用截断。

### 3. Memory 压缩层（`agentscope/agent/_react_agent.py`）

`ReActAgent.reply()` 的每轮 loop 都在调用 `_reasoning()`、真正请求模型前检查 `_compress_memory_if_needed()`：

```python
async def _compress_memory_if_needed(self) -> None:
    # 1. 取出所有未被标记 COMPRESSED 的消息
    to_compressed_msgs = await self.memory.get_memory(
        exclude_mark=_MemoryMark.COMPRESSED,
    )
    # 2. 从后向前保留最近 keep_recent 轮（tool_use/result 配对需整批保留）
    # 3. 格式化剩余消息，计算 token 数
    prompt = await self.formatter.format([Msg("system", self.sys_prompt, "system"), *to_compressed_msgs])
    n_tokens = await self.compression_config.agent_token_counter.count(prompt)
    # 4. 超过 trigger_threshold 则触发压缩
    if n_tokens > self.compression_config.trigger_threshold:
        # 5. 在历史末尾追加 compression_prompt，调用 LLM 生成 SummarySchema（结构化输出）
        res = await compression_model(compression_prompt, structured_model=SummarySchema)
        # 6. 将 summary 写入 memory._compressed_summary，旧消息标记 COMPRESSED
        await self.memory.update_compressed_summary(summary_template.format(**last_chunk.metadata))
        await self.memory.update_messages_mark(msg_ids=[...], new_mark=_MemoryMark.COMPRESSED)
```

`SummarySchema`（Pydantic BaseModel）定义了 5 个字段，每个有 max_length 约束：
- `task_overview`（300），`current_state`（300），`important_discoveries`（300），
- `next_steps`（200），`context_to_preserve`（300）

压缩后 `get_memory(prepend_summary=True)` 会将 `_compressed_summary` 包装成一条 user 消息前置到历史中，被压缩的旧消息仍留在存储中（标记为 COMPRESSED，正常检索时被 `exclude_mark` 跳过）。

`CompressionConfig` 参数：
- `enable: bool`
- `agent_token_counter: TokenCounterBase`（必须与 agent 使用的模型一致）
- `trigger_threshold: int`（token 阈值）
- `keep_recent: int = 3`（保留最近几轮不压缩）
- `compression_model / compression_formatter`（可选单独指定压缩用的模型/formatter）

### 4. Working Memory 标记系统（`agentscope/memory/_working_memory/`）

`InMemoryMemory` 存储结构为 `list[tuple[Msg, list[str]]]`，每条消息带一个 marks 列表。压缩后旧消息被打上 `COMPRESSED` 标记，新消息无标记。`get_memory()` 支持 `mark`（白名单过滤）和 `exclude_mark`（黑名单过滤），天然支持"只取未压缩消息"。

`_compressed_summary` 持久化为 StateModule 的一个状态字段，跨 session 恢复后摘要不丢。

### 5. RAG 层（`agentscope/rag/`）

RAG 是 context 扩展（而非压缩）的另一条路径，与 token 管理系统**完全解耦**，通过 `KnowledgeBase.retrieve_knowledge()` 工具函数被 agent 按需调用，返回 `ToolResponse` 注入 context：

```python
# SimpleKnowledge.retrieve()
res_embedding = await self.embedding_model([TextBlock(type="text", text=query)])
res = await self.embedding_store.search(res_embedding.embeddings[0], limit=limit, score_threshold=score_threshold)
```

`TextReader` 切分策略：char / sentence（nltk）/ paragraph，默认 chunk_size=512 字符；超长 sentence/paragraph 再按字符截断。

VDB 后端支持：Qdrant（本地 :memory: 或远程）、MilvusLite、OceanBase、MongoDB、AlibabaCloud MySQL。

---

## 关键代码路径

### Token 计数

```
TokenCounterBase.count(messages, tools)           # agentscope/token/_token_base.py
├── OpenAITokenCounter.count()                    # agentscope/token/_openai_token_counter.py
│   ├── tiktoken.encoding_for_model(model_name)
│   ├── _count_content_tokens_for_openai_vision_model()  → 图片 tile 计算
│   └── _calculate_tokens_for_tools()             → tool schema token 估算
├── AnthropicTokenCounter.count()                 # agentscope/token/_anthropic_token_counter.py
│   └── self.client.messages.count_tokens(**kwargs)
├── GeminiTokenCounter.count()                    # agentscope/token/_gemini_token_counter.py
│   └── self.client.models.count_tokens(**kwargs)
├── HuggingFaceTokenCounter.count()               # agentscope/token/_huggingface_token_counter.py
│   └── self.tokenizer.apply_chat_template(messages, tokenize=True, tools=tools)
└── CharTokenCounter.count()                      # agentscope/token/_char_token_counter.py
    └── len("\n".join([str(msg) for msg in messages]))
```

### Formatter 截断

```
TruncatedFormatterBase.format(msgs)               # agentscope/formatter/_truncated_formatter_base.py:52
├── deepcopy(msgs)
└── while True:
    ├── await self._format(msgs)                  # → OpenAIChatFormatter._format() / AnthropicFormatter._format() 等
    │   └── _group_messages() → async for (type, group)
    │       ├── "tool_sequence" → _format_tool_sequence(group)
    │       └── "agent_message" → _format_agent_message(group, is_first)
    ├── await self._count(formatted_msgs)         # → token_counter.count()
    ├── n_tokens <= max_tokens → return           # 满足限制，退出
    └── await self._truncate(msgs)                # 截断最旧 turn
        ├── 扫描 tool_call_ids 集合保证配对完整
        └── return msgs[:start_index] + msgs[i+1:]
```

### Memory 压缩

```
ReActAgent.reply(x)                               # agentscope/agent/_react_agent.py
├── await self._compress_memory_if_needed()
│   ├── memory.get_memory(exclude_mark=COMPRESSED)
│   ├── 从后向前确定 keep_recent 轮边界（保持 tool_use/result 配对）
│   ├── formatter.format([sys_msg, *to_compress]) → prompt
│   ├── compression_config.agent_token_counter.count(prompt) → n_tokens
│   ├── n_tokens > trigger_threshold？
│   │   ├── compression_formatter.format([sys, *to_compress, compression_hint]) → compression_prompt
│   │   ├── compression_model(compression_prompt, structured_model=SummarySchema)
│   │   ├── memory.update_compressed_summary(summary_template.format(**metadata))
│   │   └── memory.update_messages_mark(ids, new_mark=COMPRESSED)
└── await self._reasoning(...)                     # 内部 get_memory(prepend_summary=True)
```

### RAG 检索

```
KnowledgeBase.retrieve_knowledge(query, limit, score_threshold)   # agentscope/rag/_knowledge_base.py:77
└── SimpleKnowledge.retrieve(query, limit, score_threshold)       # agentscope/rag/_simple_knowledge.py:14
    ├── embedding_model([TextBlock(text=query)]) → res_embedding
    └── embedding_store.search(res_embedding.embeddings[0], limit, score_threshold)
        └── QdrantStore.search() / MilvusLiteStore.search() / ...  # agentscope/rag/_store/

TextReader.__call__(text)                                          # agentscope/rag/_reader/_text_reader.py:47
├── split_by="char" → [text[i:i+chunk_size] for i in range(0, len, chunk_size)]
├── split_by="sentence" → nltk.sent_tokenize() + 超长 sentence 再按 char 截断
└── split_by="paragraph" → text.split("\n") + 超长 para 再按 char 截断
    → [Document(id=sha256(text), metadata=DocMetadata(content=TextBlock, chunk_id=idx, ...))]
```

---

## 设计亮点

### 1. 三层正交，各司其职
Token 计数（精确测量）、Formatter 截断（即时 drop 旧 turn）、Memory 压缩（LLM 摘要化）三层完全解耦。截断是无损的"遗忘"，压缩是有损的"提炼"，开发者可自由选择：只用截断（无 LLM 开销）、只用压缩（更保留语义）、或两层叠加。

### 2. `CompressionConfig` 可以指定独立压缩模型
`compression_model` 和 `compression_formatter` 可以和 agent 主模型不同，典型场景是用便宜的小模型（如 gpt-4o-mini）做摘要，昂贵的大模型只做推理，节省成本。

### 3. 结构化压缩输出（`SummarySchema`）
压缩结果是强类型 Pydantic 模型，5 个字段各有 max_length 约束，通过模型的 structured output 功能强制输出 JSON。避免了自由文本摘要可能产生的格式混乱，summary 模板可自定义替换。

### 4. tool_use/result 配对安全截断
无论在 Formatter 截断（`_truncate`）还是压缩（`keep_recent` 边界计算），都用同一套 `tool_call_ids` 集合追踪机制，保证 tool_use 消息和对应 tool_result 消息永远成对出现，不会产生 API 拒绝的孤立 tool_use。

### 5. Memory 标记系统解决"压缩后仍可回溯"
被压缩的消息不是物理删除而是打 `COMPRESSED` 标记，`get_memory(exclude_mark=COMPRESSED)` 正常路径跳过，但原始消息仍在存储中可按 ID 查询（审计/调试用）。这对 Redis/SQLAlchemy/Tablestore 等持久化 backend 尤为有价值。

### 6. OpenAI 图片 token 精确实现
手动实现了官方文档的 tile 算法（2048→768 缩放→512px tiles），并针对 gpt-4o/gpt-4.1/gpt-4.1-mini/nano/o4-mini 等不同系列做了差异化参数，比简单估算更接近真实计费，避免提前截断或超出限制。

### 7. HuggingFace counter 利用 `chat_template`
直接调用 `apply_chat_template(tokenize=True, tools=tools)`，工具 schema 也纳入计数，不需要手写任何模型特定逻辑，只要模型有 `chat_template` 就能精确计数。

---

## 局限性

### 1. 截断策略单一（按 turn 从旧到新 drop）
`_truncate()` 只实现了"删除最旧完整 turn"这一策略，没有滑动窗口、重要性评分、或按 token 预算比例分配等更精细的策略。对于历史中有关键早期上下文的场景，这会造成重要信息丢失。

### 2. 压缩触发是"每轮 reply 前检查一次"
若单轮 reply 产生大量 tool_use/result（如并行调用 20 个工具），reply 内部不会再次触发压缩，可能在压缩完成后当轮又大量增长，下轮才能再次压缩。无法在一次 reply 中途 mid-flight 压缩。

### 3. 压缩不处理多模态内容（已标注 TODO）
`_compress_memory_if_needed()` 中有 `# TODO: What if the compressed messages include multimodal blocks?`，说明图片/音频消息的压缩路径未处理，实际行为未定义（可能在 LLM 处理 base64 时出现问题或直接被丢弃）。

### 4. RAG 与 token 管理完全解耦，检索结果不计入截断
`retrieve_knowledge()` 返回的内容通过工具调用注入 context，不经过 Formatter 的 `max_tokens` 截断路径（因为已经是 formatted 后的 tool_result）。若检索结果极大，可能导致实际 prompt 超出模型限制但框架不感知。

### 5. `AnthropicTokenCounter` 和 `GeminiTokenCounter` 需要网络调用
每次 format 截断循环都可能多次调用 API 计数，增加延迟（和 API 费用）。OpenAI 和 HuggingFace counter 是本地计算，无此问题。`CharTokenCounter` 虽然完全本地，但对 token 估算严重不准（字符数≠token 数，多语言偏差更大）。

### 6. TextReader 的 sentence 分割仅支持英文
`split_by="sentence"` 依赖 `nltk.sent_tokenize()`，该模块只支持英文。中文/日文等语言必须使用 `char` 或 `paragraph` 模式，而这两种模式缺乏语义感知。

### 7. `keep_recent` 以"完整 turn 数"计，不以 token 数计
`keep_recent=3` 保留"3 个完整 turn"，但每个 turn 的 token 数可能差异悬殊（一次长工具调用 vs 一次短文本回复），可能导致保留的内容仍然很多或很少，而不是精确的 token 预算控制。

---

## 来源

- 源码版本：`0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12`
- 分析深度：源码级
- 核心文件：
  - `agentscope/token/_token_base.py`、`_openai_token_counter.py`、`_anthropic_token_counter.py`、`_gemini_token_counter.py`、`_huggingface_token_counter.py`、`_char_token_counter.py`
  - `agentscope/formatter/_truncated_formatter_base.py`、`_openai_formatter.py`
  - `agentscope/agent/_react_agent.py`（`_compress_memory_if_needed`、`SummarySchema`、`CompressionConfig`）
  - `agentscope/memory/_working_memory/_base.py`、`_in_memory_memory.py`
  - `agentscope/rag/_knowledge_base.py`、`_simple_knowledge.py`、`_reader/_text_reader.py`、`_store/_store_base.py`
