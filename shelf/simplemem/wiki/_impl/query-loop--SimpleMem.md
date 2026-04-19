---
title: "query-loop——SimpleMem"
category: L2
parent: "[[query-loop]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: query-loop
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

SimpleMem 的 query loop 不是经典的 ReAct/Plan-Execute agent loop，而是专门为记忆检索设计的**意图感知检索规划循环（Intent-Aware Retrieval Planning）**。核心是 `HybridRetriever`，实现了 planning + parallel multi-view retrieval + reflection 三层嵌套循环，每次 `ask()` 调用触发一次完整的检索规划闭环。

## 架构分析

### 检索规划循环三层结构

```
ask(query)
└── [Layer 1] Planning: LLM → {q_sem, q_lex, q_sym, d}
        └── [Layer 2] Parallel Multi-View Retrieval
                ├── semantic_search(q_sem)   → R_sem
                ├── keyword_search(q_lex)    → R_lex
                └── structured_search(q_sym) → R_sym
                C_q = R_sem ∪ R_lex ∪ R_sym
        └── [Layer 3] Reflection Loop (可选, max N rounds)
                LLM: "C_q 是否足以回答 query?"
                若不足 → 生成补充查询 → 回到 Layer 2
                若充足 → 退出循环
└── AnswerGenerator.generate_answer(query, C_q)
```

### Planning 阶段（P(q, H) → {q_sem, q_lex, q_sym, d}）

LLM 解析用户意图，生成三路专用查询：
- `q_sem`：语义向量查询（自然语言，用于 embedding 相似度）
- `q_lex`：词法关键词查询（用于 BM25）
- `q_sym`：符号结构查询（时间范围、地点、人物等元数据过滤条件）
- `d`：推断的对话/事件日期范围

### Reflection 循环（Adaptive Depth）

`enable_reflection=True` 时，每轮检索后 LLM 评估结果充分性：
- 充分 → 退出，调用 AnswerGenerator
- 不充分 → 生成新查询，再次触发 Layer 2
- 硬上限：`max_reflection_rounds`（默认 2）防止无限循环

## 关键代码路径

### 完整检索循环（hybrid_retriever.py）

```python
HybridRetriever.retrieve(query: str) -> List[MemoryEntry]:
    
    # Layer 1: Planning
    if self.enable_planning:
        plan = self._plan_retrieval(query)
        # plan: {q_sem, q_lex, q_sym, date_range}
    
    # Layer 2: Parallel multi-view retrieval
    if self.enable_parallel_retrieval:
        with ThreadPoolExecutor(max_workers=self.max_retrieval_workers) as executor:
            futures = [
                executor.submit(self.vector_store.semantic_search, plan.q_sem, self.semantic_top_k),
                executor.submit(self.vector_store.keyword_search, plan.q_lex, self.keyword_top_k),
                executor.submit(self.vector_store.structured_search, plan.q_sym, self.structured_top_k),
            ]
        results = {f.result() for f in futures}
        C_q = merge_deduplicate(results)
    
    # Layer 3: Reflection loop
    for round in range(self.max_reflection_rounds):
        if not self.enable_reflection:
            break
        assessment = self._reflect(query, C_q)
        if assessment.sufficient:
            break
        supplement = self._generate_supplement_query(query, C_q, assessment)
        extra = self.vector_store.semantic_search(supplement, top_k=5)
        C_q = merge_deduplicate(C_q + extra)
    
    return C_q
```

### Planning 调用（LLM 解析意图）

```python
HybridRetriever._plan_retrieval(query: str) -> RetrievalPlan:
    messages = [
        {"role": "system", "content": "You are a retrieval planning assistant..."},
        {"role": "user", "content": build_planning_prompt(query, history_context)},
    ]
    response = self.llm_client.chat_completion(messages, temperature=0.1)
    plan = llm_client.extract_json(response)
    # 解析: {semantic_query, keyword_query, structured_filters, date_range}
    return RetrievalPlan(**plan)
```

### Answer 生成（AnswerGenerator.generate_answer）

```python
AnswerGenerator.generate_answer(query, contexts: List[MemoryEntry]) -> str:
    context_str = _format_contexts(contexts)
    messages = [
        {"role": "system", "content": "Extract concise answers from context."},
        {"role": "user", "content": _build_answer_prompt(query, context_str)},
    ]
    # 重试 3 次，解析 JSON {reasoning, answer}
    response = llm_client.chat_completion(messages, temperature=0.1)
    result = llm_client.extract_json(response)
    return result["answer"]
```

## 设计亮点

1. **Planning 前置分解查询**：一次用户查询被 LLM 分解为三种不同形态的子查询（语义/词法/符号），各自对应最优检索路径，比单一向量检索或 BM25 精度更高。

2. **Reflection 自适应深度**：不固定检索轮次，而由 LLM 动态判断结果充分性，短问题快速返回，复杂问题自动加深检索，兼顾效率和质量。

3. **并行检索消除串行瓶颈**：三路检索通过 `ThreadPoolExecutor` 并行执行，延迟约等于最慢的单路检索，而非三路之和。

4. **强制 JSON 输出 + 三次重试**：所有 LLM 调用（planning/reflection/answer）均要求 JSON 格式并内置 3 次重试，提高生产可靠性。

5. **绝对时间日期解析**：Planning 阶段使用 `dateparser` 库解析相对时间表达（"last week"）为绝对时间范围，与存储时强制绝对时间的 lossless_restatement 配合，实现精确时间过滤。

## 局限性

1. **LLM 调用次数多**：一次 `ask()` 最少触发 2 次 LLM 调用（planning + answer），开启 reflection 后可能达 4-6 次，延迟较高（通常 3-10s）。

2. **无流式输出支持的 Planning**：Planning 和 Reflection 必须等 LLM 返回完整 JSON 才能继续，不支持 streaming，用户感知延迟明显。

3. **Planning 失败的降级**：若 LLM 返回无效 Planning JSON（解析失败），代码降级到原始 query 做语义检索，丢失词法和符号检索路径，精度下降无显式告警。

4. **Reflection 轮数硬上限的保守性**：`max_reflection_rounds=2` 对复杂多跳问题可能不足，但增大会线性增加延迟和 LLM 成本，缺乏自适应退出机制（如信息增益阈值）。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
