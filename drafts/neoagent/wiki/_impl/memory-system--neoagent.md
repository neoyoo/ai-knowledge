---
title: "memory-system — neoagent"
category: L2
parent: "[[memory-system]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: memory-system
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的记忆系统是**文件+索引的极简实现**（`memory/` 共 4 文件 340 行）：`MemoryStore` 以目录为库、`MEMORY.md` 为索引 + 多个 `{topic}.md` 为主题内容；`MemoryExtractor` 在每轮 end_turn 后由 QueryLoop 触发，按 "5 个 tool call 或 4K token delta" 两阈值触发 LLM 提取（系统 prompt 约束 JSON 数组输出 filename/description/content 三字段，写盘后重建索引）；`MemoryRetriever` 用纯 lexical 评分（query 分词 vs description 子串匹配计数）取 top-5 主题注入 system prompt。无 embedding、无 SQLite、无 RAG——刻意保持"记忆 = markdown 文件夹"的可审阅、可手改性。

## 架构分析

### 四组件职责

| 组件 | 职责 | 代码位置 |
|------|------|----------|
| `MemoryStore` | 文件 I/O + 路径安全（防目录逃逸） | `store.py`（74 行） |
| `MemoryExtractor` | LLM 提取 + 索引重建 | `extractor.py`（137 行） |
| `MemoryRetriever` | lexical 检索 + 内容截断 | `retriever.py`（65 行） |
| `MemoryManager` | 协调器 + SessionState 状态 + `build_prompt_section()` | `manager.py`（64 行） |

### 存储布局

```
memory_dir/
├── MEMORY.md                     # 索引：markdown 链接列表
├── user_preferences.md           # 各主题内容文件
├── completed_tasks.md
└── project_context.md
```

`MEMORY.md` 每行形如 `- [description](filename)`，`list_topics()` 用正则 `r"-\s*\[([^\]]+)\]\(([^)]+)\)"` 解析。索引重建（`_rebuild_index`）按 `sorted(directory.glob("*.md"))` 遍历，取每文件第一条非空行（去掉 `#` 和空格）作为 160 字截断的 description。

路径安全：`_safe_path(filename)` 用 `.resolve()` + `is_relative_to(memory_dir)` 拒绝目录逃逸；加上 `extractor` 层的 `re.match(r'^[a-zA-Z0-9_\-]+\.md$', filename)` + 拒绝含 `/`/`..`/`\\`，形成两层防护。

### Extractor 触发机制

```python
_TOOL_CALLS_THRESHOLD = 5
_TOKEN_DELTA_THRESHOLD = 4000
_MAX_CONVERSATION_CHARS = 8000   # 送 LLM 前按最后 8K 字符截断

def _should_extract(tool_calls_count, token_delta) -> bool:
    return tool_calls_count >= 5 or token_delta >= 4000
```

`MemoryManager.maybe_extract(messages, current_tokens, session_state)`：
1. 首次调用 `memory_token_baseline == 0` 时设为当前 token 数（基线建立）
2. `token_delta = max(0, current_tokens - baseline)`
3. 调 `extractor.extract(messages, tool_calls_count, token_delta)`
4. 若 stored > 0，重置 `memory_tool_calls = 0` 和 `memory_token_baseline = current_tokens`（新周期起点）

`QueryLoop` 在每 end_turn 分支自动调用（`core/loop.py:241-255`）：

```python
if self._memory_manager:
    current_tokens = self._compressor.estimate_tokens(msgs)
    tool_calls_before = session_state.memory_tool_calls
    triggered, items_stored = await self._memory_manager.maybe_extract(
        msgs, current_tokens, session_state=session_state
    )
    self._bus.emit(MemoryExtractEvent(
        triggered=triggered,
        tool_calls=tool_calls_before,
        token_delta=token_delta,
        items_stored=items_stored,
    ))
```

tool_use 分支结束时（循环即将进下一轮）调 `record_tool_calls(len(tool_calls), session_state)` 累计。

### 提取 Prompt 约束

```
You are a memory extraction assistant for an AI agent.
Given a conversation, extract long-term information worth remembering across sessions.

Output a JSON array. Each element has exactly these keys:
  "filename"    — short snake_case name ending in .md
  "description" — one sentence ≤160 chars, used as retrieval index
  "content"     — full markdown content to store

Only extract information that is:
- Stable across sessions (not ephemeral task details)
- Useful to recall in future conversations
- Not derivable from reading the code or documentation

If nothing is worth extracting, return an empty array: []

Return ONLY valid JSON. No explanation, no markdown fences.
```

容错：解析前 strip ``` 代码围栏；`json.loads` 抛异常时 `return 0` 不报错（LLM 偶发格式出错不阻塞主流程）；每条 item 校验 filename 正则 + 非空字段，不合格直接 skip。

### Retriever 检索

```python
def retrieve(self, query: str | None = None) -> str:
    index = self._store.read_index()
    if query is None:
        return f"# Memory\n{index}"       # 无 query：只注入索引
    
    topics = self._store.list_topics()
    query_words = set(query.lower().split())
    scored = [(sum(1 for w in query_words if w in desc.lower()), fname, desc) for fname, desc in topics]
    scored.sort(key=lambda x: -x[0])
    top = [x for x in scored if x[0] > 0] or scored  # 无命中则取全部前 N
    top = top[:5]                                    # max_files=5
    # 每条取前 2000 字符
```

返回格式：

```
# Memory
- [description1](file1.md)
- [description2](file2.md)
...

## description1
（file1 前 2000 字符）

## description2
（file2 前 2000 字符）
```

### Prompt 注入

`NeoAgent.enable_memory(memory_dir, project_key)` 里注入：

```python
self._prompt_builder.add_section(PromptSection(
    name="memory",
    content=lambda: memory_manager.build_prompt_section(),
    priority=5,
    is_static=False,
))
```

默认 `memory_dir = ~/.neoagent/memory/{hash(cwd)[:8]}`（cwd 的 sha256 前 8 字符）——同一项目目录跨 session 自然共享记忆库；显式 `project_key` 可覆盖。`build_prompt_section(query=None)` 默认只注入索引（不加载主题文件），要按查询检索需要应用层主动传 query。

### 关键代码路径

- `neoagent/memory/store.py:8-74` — `MemoryStore`（`_safe_path`、`read_topic`、`write_topic`、`list_topics`）
- `neoagent/memory/extractor.py:17-33` — `_EXTRACT_SYSTEM` prompt 模板
- `neoagent/memory/extractor.py:58-120` — `extract()` LLM 调用 + 校验 + 写盘
- `neoagent/memory/extractor.py:122-137` — `_rebuild_index` 索引重建
- `neoagent/memory/retriever.py:28-65` — lexical 评分 retrieve
- `neoagent/memory/manager.py:28-61` — `maybe_extract` 基线管理
- `neoagent/agent.py:179-204` — `enable_memory()` 默认路径 + prompt section 注入
- `neoagent/core/loop.py:241-255` — 循环触发点（end_turn 分支）

## 设计亮点

- **文件夹即数据库**：整个记忆系统没有 SQLite/LanceDB/FAISS 任何外部依赖，每条记忆就是一个可 `cat`、可 `vim` 编辑的 markdown。这是**"可审阅记忆"**的极端版本——用户可以手动添加/删除/修改记忆文件，下一轮自动生效（lambda content）。
- **两阈值 OR 触发**（5 tool calls 或 4K token delta）：比固定"每 N 轮触发一次"灵活——短对话工具密集也能触发、长对话纯文本也能触发。
- **baseline reset 防重复触发**：成功抽取后重置 `memory_token_baseline = current_tokens`，避免同一批对话多次触发记忆提取浪费 token。
- **项目级默认路径**：`hash(cwd)[:8]` 让同项目不同 session 自然共享记忆库，又天然隔离不同项目。
- **`Only extract information that is stable across sessions` 的提取 prompt 自律**：通过 prompt 约束 LLM 只提取长期有效信息（用户偏好、项目上下文），避免把"刚才算了 3+5=8"这种短期事实写入库——prompt engineering 做护栏。
- **查询可选的 retrieve**：`query=None` 只给索引，让 agent 自己决定要不要深入查某主题；`query=...` 给 top-5 内容——两种注入模式并存。

## 局限性

- **Lexical 检索弱**：单词子串匹配，同义词/同概念不同表达无法检索（"用户偏好"和"user preferences" 不匹配）。无 embedding/向量检索让这个系统在英文项目上还行，中英混合场景会经常错检。
- **无更新语义**：新主题覆盖旧主题（`write_topic` = `path.write_text`）——如果 LLM 提取时生成了和已有主题重名的 filename，旧内容直接丢失。无 append/merge/diff 机制。
- **提取 prompt 没拿到 session state**：LLM 不知道已有哪些主题，可能重复提取相同信息（"user prefers dark mode" 被多次写入 user_prefs.md 覆盖——好的一面是幂等，坏的一面是无法迭代精化）。改进方案：在 prompt 里注入当前索引。
- **检索有 2000 字符/文件的硬截断**：可能从大主题文件中途切断，语义完整性丢失。
- **无跨项目记忆**：`memory_dir` 一 session 只绑定一个目录，无法"同时用项目 A 和项目 B 的记忆"——和 mem0 等专业记忆系统的多 namespace 能力差距明显。
- **索引重建是 O(N × file_size)**：每次存储触发后扫所有文件读第一行，文件多时慢——可以用专门的 index.json 替代但失去了"手动可编辑"属性。
- **没有 TTL / 过期清理**：记忆文件只增不减，长时间积累后可能充满过时信息，`list_topics` 遍历开销线性增长。
- **`maybe_extract` 里 `else` 分支 fallback：no session_state → no-op**：如果调用者没 session_state，记忆提取永远不触发——manager.py:60 明写 `Fallback: no session_state — no-op`。这意味着记忆强耦合 Session。
- **MemoryExtractEvent 的 `filenames` 字段未填充**：`MemoryExtractEvent(triggered, tool_calls, token_delta, items_stored, filenames=())` 被发射时 `filenames=()`（loop.py:249 注释 "filenames not populated yet"）——Observer 只能知道"存了 N 条"，不知道"存了哪些"。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/memory/store.py`, `extractor.py`, `retriever.py`, `manager.py`, `neoagent/agent.py:179-204`, `neoagent/core/loop.py:241-255`
