---
title: "context-management——SimpleMem"
category: L2
parent: "[[context-management]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: context-management
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

SimpleMem 的 context-management 体现在两个层面：（1）写入侧——通过语义结构化压缩将长对话窗口压缩为紧凑记忆单元，大幅降低未来注入 context 的 token 消耗；（2）跨会话注入侧（cross/ 层）——`context_injector.py` 负责 token 预算管控，将历史 session 摘要和观察以受控 token 量注入新会话的 system prompt。

## 架构分析

### 写入侧压缩（Token 压缩策略）

`core/memory_builder.py` 实现 **Semantic Structured Compression（Section 3.1）**：

核心思路：不保存原始对话文本，而是让 LLM 提取"无损重述（lossless_restatement）"——一句话包含完整主谓宾+绝对时间+地点，比原始对话文本短得多，可直接作为 context 注入。

窗口大小由 `config.WINDOW_SIZE` 控制，`OVERLAP_SIZE` 保证跨窗口语义连续：
```python
self.step_size = max(1, self.window_size - self.overlap_size)
```

### 跨会话 Context 注入（Token 预算管控）

`cross/context_injector.py` 的 `ContextInjector` 类实现 token budgeting：

1. 从 SQLite 取最近 N 个 session 摘要（按重要性排序）
2. 从 LanceDB 取语义匹配的 CrossObservation
3. 按 token 预算上限逐条填充，超出预算截止
4. 拼装为结构化 `ContextBundle`，注入 system prompt 前缀

### ContextBundle 数据结构（cross/types.py）

```python
class ContextBundle(BaseModel):
    session_summaries: List[SessionSummary]      # 历史会话摘要列表
    observations: List[CrossObservation]          # 结构化观察（决策/bugfix/特性等）
    semantic_matches: List[str]                   # 语义相关记忆片段
    total_tokens: int                             # 估算 token 总量
    formatted_context: str                        # 已格式化的注入文本
```

## 关键代码路径

### 写入时压缩（降低未来 context token 消耗）

```python
# core/memory_builder.py
MemoryBuilder._generate_memory_entries(dialogues: List[Dialogue])
    └── _build_extraction_prompt(dialogue_text, dialogue_ids, context)
            # 关键约束：
            # "Force Disambiguation: PROHIBIT pronouns and relative time"
            # → 生成: "Alice suggested at 2025-11-15T14:30:00 to meet Bob at Starbucks..."
    └── llm_client.chat_completion(messages, temperature=0.1)
    └── _parse_llm_response(response) → List[MemoryEntry]
            # MemoryEntry.lossless_restatement: 1句话完整描述，无歧义
```

### 跨会话 Context 注入路径

```python
# cross/session_manager.py → cross/context_injector.py
CrossSessionOrchestrator.session_start(tenant_id, content_session_id, project, user_prompt)
    └── ContextInjector.build_context_bundle(user_prompt, token_budget)
            ├── SQLiteStorage.get_recent_summaries(tenant_id, limit=N)
            ├── CrossSessionVectorStore.search(user_prompt, top_k)  # 语义匹配
            ├── token_budget 逐条累加，超出截止
            └── format_context_bundle() → formatted_context (str)
    └── HookResult(context_bundle=bundle)
```

### answer_generator 中的 context 组装

```python
# core/answer_generator.py
AnswerGenerator._format_contexts(contexts: List[MemoryEntry]) -> str
    # 每条 MemoryEntry 格式化为:
    # [Context N]
    # Content: {lossless_restatement}
    # Time: {timestamp}
    # Location: {location}
    # Persons: {persons}
    # Topic: {topic}
```

## 设计亮点

1. **压缩即注入优化**：写入阶段的语义压缩不仅是存储优化，更是 context window 优化——lossless_restatement 平均比原始对话短 3-5x，使同等 token 预算能注入更多历史信息。

2. **无损信息保证**：extraction prompt 的"Complete Coverage"要求（"Generate enough memory entries to ensure ALL information is captured"）确保压缩后不丢关键信息，与一般摘要的有损压缩不同。

3. **Token 预算硬截止**：ContextInjector 的 token_budget 机制是硬上限而非软建议，避免在长期使用后 context 无界增长导致模型性能下降。

4. **三类 context 分层注入**：摘要（session-level）+ 观察（structured facts）+ 语义匹配（point-level），三层粒度互补，高层摘要提供全局理解，语义匹配提供具体细节。

## 局限性

1. **Token 计数精度**：代码中 token 估算使用字符数/4 的近似公式，对中文、代码等非英语内容误差较大，可能导致实际注入量超出或远低于预算。

2. **压缩质量 LLM 依赖**：lossless_restatement 质量完全取决于 LLM 能力，弱模型（如小参数量本地模型）可能产生有损压缩或歧义重述。

3. **文本版无跨会话 context**：core/ 的文本版 `SimpleMemSystem` 无 `ContextInjector` 集成，跨会话 context 注入仅在 cross/ 层可用，两版本功能不对称。

4. **Overlap 增加冗余**：滑动窗口的 overlap_size 保证语义连续性，但同一对话可能被处理两次（出现在两个窗口中），产生内容相近的 MemoryEntry，检索时增加噪声。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
