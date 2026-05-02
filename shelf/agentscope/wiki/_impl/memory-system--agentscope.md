---
title: "memory-system——agentscope"
category: L2
parent: "[[memory-system]]"
source: agentscope
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "memory-system"
created: "2026-04-15"
confidence: high
---

## 概述

AgentScope 将记忆系统分为两个正交维度：**Working Memory**（对话历史存储）和 **Long-Term Memory**（跨会话持久记忆）。前者提供四种可互换的后端（内存/Redis/SQLAlchemy/Tablestore），统一以 `MemoryBase` 接口操作，并内置 `_compressed_summary` 压缩摘要机制；后者提供两条技术路线（mem0 向量语义路线 / ReMe 任务-个人-工具三分路线），均以工具函数 `record_to_memory` / `retrieve_from_memory` 暴露给 LLM 自主调用。两层记忆通过 `StateModule` 基类统一支持序列化/反序列化，天然适配分布式部署。

---

## 架构分析

### 两层记忆架构

```
Working Memory (短期)                Long-Term Memory (长期)
─────────────────────────────        ───────────────────────────────
MemoryBase (StateModule)             LongTermMemoryBase (StateModule)
  ├── InMemoryMemory                   ├── Mem0LongTermMemory
  ├── RedisMemory                      ├── ReMePersonalLongTermMemory
  ├── AsyncSQLAlchemyMemory            ├── ReMeTaskLongTermMemory
  └── TablestoreMemory                 └── ReMeToolLongTermMemory

核心操作: add / get_memory /          核心操作: record / retrieve
         delete / delete_by_mark /             record_to_memory /
         update_messages_mark                  retrieve_from_memory
         _compressed_summary
```

两层之间没有自动关联——开发者需手动在 agent reply 逻辑中串联：先从 LTM retrieve 注入 system prompt，reply 后再 record 进 LTM。

### Working Memory：Mark 标记系统

Working Memory 最核心的设计是 **Mark**（标记）：每条消息入库时可附带一个或多个字符串标签（`marks: str | list[str]`），检索时可按 mark 过滤或排除。这让同一个 memory 实例可以同时服务多个对话角色/会话流而不互相污染。

`InMemoryMemory` 内部结构：

```python
self.content: list[tuple[Msg, list[str]]] = []
```

每条消息存为 `(Msg, marks_list)` 元组，检索时列表推导过滤。

`RedisMemory` 的 Mark 实现更复杂，维护了专门的索引 Key：

```
user_id:{uid}:session:{sid}:messages          # 全量消息 ID 顺序列表 (List)
user_id:{uid}:session:{sid}:msg:{msg_id}      # 消息 payload (String)
user_id:{uid}:session:{sid}:mark:{mark}       # 某标记下的消息 ID (List)
user_id:{uid}:session:{sid}:marks_index       # 所有 mark 名称集合 (Set)
```

`marks_index` 是关键优化：避免了查找所有 mark 时需要全量 `SCAN`。同时支持旧数据迁移（`_scan_and_migrate_marks`），向后兼容。

### Working Memory：压缩摘要（Compressed Summary）

`MemoryBase` 基类内置 `_compressed_summary: str`，通过 `register_state` 参与序列化。调用 `get_memory(prepend_summary=True)` 时，若摘要非空，它会作为第一条 `user` 角色的 Msg 预置于返回列表头部。

这是一个「手动压缩」方案——AgentScope 不自动触发压缩，由开发者调用 `update_compressed_summary(summary)` 写入。框架不提供内置的压缩触发器或 token 计数策略。

### Long-Term Memory：mem0 路线

`Mem0LongTermMemory` 是对 `mem0.AsyncMemory` 的适配封装：

- 注册自定义 `agentscope` provider 到 mem0 的 `LlmFactory` / `EmbedderFactory`，使 AgentScope 自有的 `ChatModelBase` / `EmbeddingModelBase` 可作为 mem0 的底层引擎。
- 支持三维标识符（`agent_id` / `user_id` / `run_id`），映射到 mem0 的记忆隔离维度。
- `record_to_memory` 实现了**三级降级策略**：user 角色消息 → assistant 角色消息 → `infer=False` 直接写入，确保内容一定落库。
- `retrieve_from_memory` 对每个 keyword 并发发起 `mem0.search`（`asyncio.gather`），支持语义向量 + 图谱关系（`relations`）双路返回。

### Long-Term Memory：ReMe 路线

`ReMe` 是阿里 ModelScope 开发的记忆库，AgentScope 通过 `ReMeLongTermMemoryBase` 适配，提供三个专用子类：

| 类 | ReMe flow name | 用途 |
|---|---|---|
| `ReMePersonalLongTermMemory` | `summary_personal_memory` / `retrieve_personal_memory` | 用户偏好、个人事实 |
| `ReMeTaskLongTermMemory` | `summary_task_memory` / `retrieve_task_memory` | 任务执行经验、步骤学习 |
| `ReMeToolLongTermMemory` | `summary_tool_memory` / `retrieve_tool_memory` | 工具调用模式、使用指南 |

ReMe 要求 `async with memory:` 上下文管理器，`__aenter__` 启动 ReMe App 上下文，`__aexit__` 清理。若未进入上下文直接调用，会抛 `RuntimeError`。

### RAG 子系统

`rag/` 是独立于 memory 的知识库模块，面向静态文档检索：

```
KnowledgeBase
  ├── embedding_model: EmbeddingModelBase
  └── embedding_store: VDBStoreBase
        ├── MilvusLiteStore
        ├── QdrantStore
        ├── MongoDBStore
        ├── OceanBaseStore
        └── AlibabaCloudMySQLStore
```

`SimpleKnowledge.retrieve` 调用链：`query` → `embedding_model(TextBlock)` → `embedding_store.search(query_embedding, limit)` → `list[Document]`。

`KnowledgeBase` 提供 `retrieve_knowledge` 作为 ToolResponse 包装，方便直接注册为 agent 工具。RAG 模块没有与 Working Memory / LTM 的直接依赖，三者正交，由开发者自行决定在 agent 逻辑中如何组合。

---

## 关键代码路径

### Working Memory 写入（Redis 实现）

```python
# agentscope/memory/_working_memory/_redis_memory.py

async def add(self, memories: Msg | list[Msg] | None,
              marks: str | list[str] | None = None,
              skip_duplicated: bool = True, **kwargs) -> None:
    # 1. 查现有 msg_ids 去重
    existing_msg_ids = await self._client.lrange(self._get_session_key(), 0, -1)
    messages_to_add = [m for m in memories if m.id not in existing_msg_ids_set]

    # 2. Pipeline 原子写入
    pipe = self._client.pipeline()
    await pipe.rpush(self._get_session_key(), *[m.id for m in messages_to_add])  # 顺序 List
    for m in messages_to_add:
        await pipe.set(self._get_message_key(m.id), json.dumps(m.to_dict()))     # payload String
        for mark in mark_list:
            await pipe.rpush(self._get_mark_key(mark), m.id)                     # mark List
            await pipe.sadd(self._get_marks_index_key(), mark)                    # marks_index Set
    await self._refresh_session_ttl(pipe=pipe)
    await pipe.execute()
```

### Working Memory 读取（带 mark 过滤）

```python
async def get_memory(self, mark=None, exclude_mark=None,
                     prepend_summary=True, **kwargs) -> list[Msg]:
    if mark is None:
        msg_ids = await self._client.lrange(self._get_session_key(), 0, -1)
    else:
        msg_ids = await self._client.lrange(self._get_mark_key(mark), 0, -1)

    # exclude_mark 过滤
    if exclude_mark:
        exclude_ids = set(await self._client.lrange(self._get_mark_key(exclude_mark), 0, -1))
        msg_ids = [_ for _ in msg_ids if _ not in exclude_ids]

    # mget 批量读取，避免 N+1
    msg_keys = [self._get_message_key(msg_id) for msg_id in msg_ids]
    msg_data_list = await self._client.mget(msg_keys)
    messages = [Msg.from_dict(json.loads(d)) for d in msg_data_list if d]

    # 摘要预置
    if prepend_summary and self._compressed_summary:
        return [Msg("user", self._compressed_summary, "user"), *messages]
    return messages
```

### mem0 长期记忆写入（三级降级）

```python
# agentscope/memory/_long_term_memory/_mem0/_mem0_long_term_memory.py

async def record_to_memory(self, thinking: str, content: list[str]) -> ToolResponse:
    content = [thinking] + content
    # Strategy 1: user role
    results = await self._mem0_record([{"role": "user", "content": "\n".join(content)}])

    # Strategy 2: assistant role fallback
    if results["results"] == []:
        results = await self._mem0_record([{"role": "assistant", "content": ...}])

    # Strategy 3: direct write, no inference
    if results["results"] == []:
        results = await self._mem0_record([...], infer=False)

    return ToolResponse(...)

async def _mem0_record(self, messages, memory_type=None, infer=True, **kwargs) -> dict:
    return await self.long_term_working_memory.add(
        messages=messages,
        agent_id=self.agent_id, user_id=self.user_id, run_id=self.run_id,
        memory_type=memory_type or self.default_memory_type,
        infer=infer, **kwargs,
    )
```

### mem0 并发检索

```python
async def retrieve(self, msg: Msg | list[Msg], limit=5) -> str:
    msg_strs = [json.dumps(_.to_dict()["content"]) for _ in msg]
    search_coroutines = [
        self.long_term_working_memory.search(query=item,
            agent_id=self.agent_id, user_id=self.user_id, run_id=self.run_id,
            limit=limit)
        for item in msg_strs
    ]
    search_results = await asyncio.gather(*search_coroutines)  # 并发检索
    results = []
    for result in search_results:
        results.extend([memory["memory"] for memory in result["results"]])
        if "relations" in result:
            results.extend(self._format_relations(result))  # 图谱关系拼接
    return "\n".join(results)
```

### RAG 检索链

```python
# agentscope/rag/_simple_knowledge.py

async def retrieve(self, query: str, limit=5, score_threshold=None) -> list[Document]:
    res_embedding = await self.embedding_model([TextBlock(type="text", text=query)])
    res = await self.embedding_store.search(
        res_embedding.embeddings[0], limit=limit, score_threshold=score_threshold
    )
    return res
```

---

## 设计亮点

### 1. Mark 标记系统——轻量级多上下文隔离

大多数记忆系统只有「全量历史」或「按会话隔离」两档。AgentScope 的 mark 系统提供了第三档：**同一会话内的语义分区**。例如可以给「工具调用历史」打 `tool_calls` mark，给「规划步骤」打 `planning` mark，不同场景下 `get_memory(mark=...)` 精确取出所需子集，而不是每次传入完整历史。这对 multi-agent 编排中工具链和主对话流分离特别有价值。

Redis 实现还额外维护了 `marks_index`（Redis Set），避免了扫描全量 key 来获取所有 mark 名称的 O(N) 问题，是实际生产级的细节处理。

### 2. 记忆三分法（ReMe 路线）

将长期记忆拆分为 Personal / Task / Tool 三个独立实例，各自有专属的 prompt flow，而不是一个通用向量库搞定一切。这个设计承认了不同类型记忆的检索策略本质不同：个人偏好需要语义相似，任务经验需要结构化步骤检索，工具指南需要基于工具名称精确匹配。每个子类有独立的 `record_to_memory` docstring 说明何时应该记录，直接作为 LLM 工具的 description 使用。

### 3. 三级降级写入（mem0 路线）

`record_to_memory` 的三级策略（user message → assistant message → infer=False）是实用的容错设计。mem0 的 LLM 推断有时因为 prompt 对角色的偏好导致某些内容无法提取，三级降级保证了「原始内容一定能落库」，而不是静默失败。

### 4. StateModule 统一序列化协议

`MemoryBase` 和 `LongTermMemoryBase` 都继承 `StateModule`，通过 `register_state(field_name)` + `state_dict()` / `load_state_dict()` 实现统一序列化接口。`InMemoryMemory` 实现了向后兼容逻辑：旧版只存 `dict`，新版存 `[msg_dict, marks]` 二元组，`load_state_dict` 通过 `isinstance(item, dict)` 分支兼容两种格式。

### 5. Redis 滑动 TTL

`RedisMemory` 支持 `key_ttl` 参数，每次读写操作后调用 `_refresh_session_ttl` 刷新全部会话相关 Key 的过期时间，实现滑动窗口语义（"最后一次活跃起 N 秒后自动清理"），而非固定过期。

---

## 局限性

### 1. 两层记忆无自动联动

Working Memory 和 Long-Term Memory 完全独立，框架不提供任何自动触发机制（如「对话轮数超过 N 时自动压缩并写入 LTM」）。开发者必须手动在 agent reply 逻辑中串联两层，实际上这增加了使用复杂度，很容易被忽略。

### 2. 压缩摘要机制残缺

`_compressed_summary` 字段存在，但框架没有提供：
- 自动压缩触发器（基于 token 数 / 消息数）
- 内置压缩 LLM 调用
- 压缩版本管理

开发者需要完全自己实现「何时压缩、如何压缩」，这个字段更像一个预留槽位而非完整功能。

### 3. ReMe 对 DashScope/OpenAI 的强绑定

`ReMeLongTermMemoryBase.__init__` 中对模型类型有硬编码判断：

```python
if isinstance(model, DashScopeChatModel): ...
elif isinstance(model, OpenAIChatModel): ...
else:
    raise ValueError(...)  # 其他模型直接报错
```

这意味着 AgentScope 自有的其他 LLM provider（如 Anthropic、Ollama 等）无法使用 ReMe 路线的 LTM，只能用 mem0 路线。

### 4. RAG 与记忆系统完全隔离

`rag/` 模块和 `memory/` 模块没有任何耦合。现实场景中「从知识库中召回文档片段并注入上下文」与「从长期记忆中召回对话摘要」常常需要协同排序，但 AgentScope 不提供统一的召回融合接口。

### 5. Tablestore/SQLAlchemy 实现的 `delete_by_mark` 未在基类统一

`MemoryBase` 将 `delete_by_mark` 声明为 `raise NotImplementedError`（非抽象方法），意味着某些后端实现可能静默缺失这个功能。调用方无法通过类型检查确认某个实现是否支持，只能运行时发现。

### 6. mem0 版本兼容问题

代码中有显式的版本分支判断（`mem0ai <= 0.1.115`），说明 mem0 的 API 曾发生破坏性变更，当前适配逻辑较脆，未来 mem0 升级时可能再次失效。`suppress_mem0_logging=True` 的默认值也表明 Qdrant validation 错误是已知问题但未根本解决。

---

## 来源

- 源码版本：`0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12`
- 分析深度：源码级
- 核心文件：
  - `agentscope/memory/_working_memory/_base.py`
  - `agentscope/memory/_working_memory/_in_memory_memory.py`
  - `agentscope/memory/_working_memory/_redis_memory.py`
  - `agentscope/memory/_working_memory/_sqlalchemy_memory.py`
  - `agentscope/memory/_working_memory/_tablestore_memory.py`
  - `agentscope/memory/_long_term_memory/_long_term_memory_base.py`
  - `agentscope/memory/_long_term_memory/_mem0/_mem0_long_term_memory.py`
  - `agentscope/memory/_long_term_memory/_reme/_reme_long_term_memory_base.py`
  - `agentscope/memory/_long_term_memory/_reme/_reme_personal_long_term_memory.py`
  - `agentscope/memory/_long_term_memory/_reme/_reme_task_long_term_memory.py`
  - `agentscope/rag/_knowledge_base.py`
  - `agentscope/rag/_simple_knowledge.py`
  - `agentscope/rag/_store/_store_base.py`
