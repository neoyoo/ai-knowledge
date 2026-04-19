---
title: "memory-system——SimpleMem"
category: L2
parent: "[[memory-system]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: memory-system
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

SimpleMem 实现了一套面向 LLM Agent 的高效终身记忆系统，核心是三阶段管道：语义结构化压缩（Stage 1）、在线语义合成（Stage 2）、意图感知检索规划（Stage 3）。系统有两个版本：轻量文本版（core/ + database/）和多模态版（OmniSimpleMem/omni_memory/），通过 simplemem_router.py 统一路由。

## 架构分析

### 整体架构

```
用户输入 (dialogue)
    │
    ▼
SimpleMemSystem (main.py)
    ├── MemoryBuilder (core/memory_builder.py)        ← Stage 1+2: 压缩+合成
    │       ├── sliding window segmentation
    │       ├── LLM semantic density gating
    │       └── VectorStore.add_entries()
    ├── HybridRetriever (core/hybrid_retriever.py)   ← Stage 3: 检索规划
    │       ├── LLM planning: P(q, H) → {q_sem, q_lex, q_sym, d}
    │       ├── parallel multi-view retrieval
    │       └── reflection loop
    └── AnswerGenerator (core/answer_generator.py)   ← 最终合成
            └── LLM synthesis from C_q
```

### 数据模型

`models/memory_entry.py` 定义两个核心模型：
- `Dialogue`: 对话输入单元（speaker, content, timestamp, dialogue_id）
- `MemoryEntry`: 压缩后的记忆单元，包含 lossless_restatement / keywords / timestamp / location / persons / entities / topic

### 存储层

`database/vector_store.py` 封装 LanceDB，实现：
- 向量语义检索（cosine similarity）
- BM25 关键词检索
- 结构化元数据过滤（时间、地点、人物）

### 统一路由器

`simplemem_router.py` 使用 Registry 模式（受 HuggingFace AutoModel 启发），通过 `_Backend` 描述符惰性加载两个后端：
- `mode="text"` → `SimpleMemSystem`
- `mode="omni"` → `OmniSimpleMem`

### 多模态版本（OmniSimpleMem）

`OmniSimpleMem/omni_memory/` 扩展支持 text/image/audio/video，新增：
- `core/mau.py`：多模态感知单元（Multimodal Awareness Unit）
- `retrieval/pyramid_retriever.py`：金字塔式分层检索
- `routing/router.py`：模态路由
- `evolution/`：记忆演化（时间衰减+重要性更新）
- `graph/`：知识图谱层

## 关键代码路径

### 写入路径（add_dialogue → 压缩存储）

```python
# main.py
SimpleMemSystem.add_dialogue(speaker, content, timestamp)
    └── MemoryBuilder.add_dialogue(dialogue, auto_process=True)
            ├── [buffer >= window_size] → process_window()
            │       ├── window = dialogue_buffer[:window_size]
            │       ├── dialogue_buffer = dialogue_buffer[step_size:]  # overlap保留
            │       ├── _generate_memory_entries(window)
            │       │       ├── _build_extraction_prompt(dialogue_text, ids, context)
            │       │       └── llm_client.chat_completion() → JSON → MemoryEntry[]
            │       └── vector_store.add_entries(entries)
            └── [大批量] → add_dialogues_parallel()
                    └── ThreadPoolExecutor._process_windows_parallel()
```

### 查询路径（ask → 多视图检索 → 生成答案）

```python
# main.py
SimpleMemSystem.ask(query)
    └── HybridRetriever.retrieve(query)
            ├── [enable_planning] LLM planning → {q_sem, q_lex, q_sym, d}
            ├── parallel retrieval (ThreadPoolExecutor):
            │       ├── vector_store.semantic_search(q_sem, top_k)    # R_sem
            │       ├── vector_store.keyword_search(q_lex, top_k)     # R_lex
            │       └── vector_store.structured_search(q_sym, top_k)  # R_sym
            ├── merge: C_q = R_sem ∪ R_lex ∪ R_sym
            └── [enable_reflection] reflection loop (max N rounds)
                    └── LLM判断是否需要补充检索
    └── AnswerGenerator.generate_answer(query, C_q)
            └── LLM synthesis → JSON {reasoning, answer}
```

### 路由器工厂

```python
# simplemem_router.py
simplemem.create(mode="text", **kwargs)
    └── _REGISTRY[mode]._load()  # 惰性导入
            └── _Backend.init(mode, module_path, class_name)
```

## 设计亮点

1. **隐式语义密度门控（Φ_gate）**：不依赖显式规则过滤低信息窗口，而是让 LLM 自主决定哪些内容值得压缩为记忆单元，天然适应对话密度变化。

2. **强制消歧提示工程**：Extraction prompt 明确禁止代词和相对时间（"yesterday"/"last week"），强制生成"主语+谓语+宾语+绝对时间"的无歧义表达，解决记忆跨会话引用时指代不清的痛点。

3. **滑动窗口 + overlap 保留**：`step_size = window_size - overlap_size`，相邻窗口共享 overlap_size 条对话，保证窗口边界处的语义连续性，同时避免重复处理。

4. **三视图并行检索**：语义（vector cosine）+ 词法（BM25）+ 符号（结构化元数据过滤）三路并行，各自独立异步执行后取并集，兼顾语义模糊匹配和精确关键词/时间查询。

5. **反射式检索循环**：Reflection 机制让 LLM 判断已检索结果是否充分，不足时自动生成补充查询，动态调整检索深度，比固定 top-k 更鲁棒。

6. **Registry 路由器模式**：`_Backend` 描述符惰性加载，新增后端只需注册元数据，不污染主模块导入链，依赖隔离彻底。

## 局限性

1. **LLM 依赖深重**：写入（压缩）和读取（规划+反射+生成答案）都依赖 LLM 调用，高频写入场景延迟和成本显著，不适合实时流式对话。

2. **并行处理的顺序依赖问题**：`_process_windows_parallel` 中 `previous_entries`（用于去重 context）是跨 worker 共享的全局状态，并行窗口实际上读取的是同一个快照，无法保证窗口间去重的时序一致性。

3. **向量存储无分片**：LanceDB 单表存储所有记忆，缺少租户隔离或分片机制（cross/ 层的多租户隔离靠 SQLite 的 tenant_id 字段，文本版无此机制）。

4. **多模态版本耦合度低**：OmniSimpleMem 和核心文本版是两套独立实现，router 做了封装但接口不完全兼容（omni 版用 `add_text`/`query`，文本版用 `add_dialogue`/`ask`）。

5. **无记忆淘汰策略（文本版）**：文本版 VectorStore 无 TTL 或重要性衰减，记忆只增不减，长期运行后检索噪声会持续增大（cross/ 层的 consolidation.py 有衰减，但文本版未集成）。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
