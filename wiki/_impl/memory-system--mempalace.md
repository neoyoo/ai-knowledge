---
title: "Memory System — MemPalace"
category: L2
parent: "[[memory-system]]"
source: mempalace
source_version: "latest (March 2026)"
confidence: high
created: 2026-04-09
updated: 2026-04-09
relations:
  - target: "[[memory-system]]"
    type: implements
  - target: "[[memory-system--hermes-agent]]"
    type: contrasts_with
---

## 概述

MemPalace 是目前所有对比项目中**唯一坚持 raw verbatim storage 的记忆系统**：永远存原文，永远不做 LLM 摘要/提取，完全依赖 ChromaDB 向量语义搜索保证检索质量。其设计哲学是对行业主流（Mem0/Mastra/Supermemory 都用 LLM 提取记忆）的直接反驳——"simpler approach wins because it doesn't lose information"。LongMemEval R@5 96.6% 的零 API 成本得分是这一哲学的实证支撑。

---

## 架构概览

```
用户文件/对话
     │
     ▼
normalize.py          ← 格式归一（6 种来源格式 → 统一 transcript）
     │
     ▼
miner.py / convo_miner.py  ← chunking + room 自动分类
     │
     ▼
ChromaDB ("mempalace_drawers")   ← 语义向量索引，verbatim 存储
     │
     ├── layers.py (MemoryStack)    ← 4 层读取接口（L0/L1/L2/L3）
     ├── searcher.py                ← 语义搜索 + metadata 过滤
     ├── palace_graph.py            ← Room 图遍历（Wings 作节点）
     ├── knowledge_graph.py         ← SQLite 时态 KG（独立于向量库）
     └── mcp_server.py              ← 19 个 MCP 工具（Claude Code 集成）

~/.mempalace/
     ├── palace/          ← ChromaDB 持久化数据
     ├── knowledge_graph.sqlite3   ← 时态 KG
     ├── identity.txt     ← L0 身份文本（用户手写）
     ├── config.json      ← 配置（palace_path/wings/hall_keywords）
     ├── aaak_entities.md ← AAAK 实体注册表（onboarding 生成）
     └── critical_facts.md ← 启动前 bootstrap 事实

Palace 空间隐喻:
  Wing  ← 人/项目维度（顶层，如 wing_code / family / health）
  Hall  ← 话题大类（hall_facts / hall_events / hall_decisions）
  Room  ← 具体主题 slug（如 chromadb-setup / riley-college-apps）
  Drawer ← 单条 verbatim 内容单元（ChromaDB 的 document）
```

---

## 各模块深度分析

### 1. layers.py — 4 层记忆栈（核心架构）

4 层的本质是**按需渐进加载**，最小化 wake-up token 开销：

| 层 | token 预算 | 触发时机 | 内容 |
|---|---|---|---|
| L0 Identity | ~100 tokens | 始终加载 | `~/.mempalace/identity.txt`，用户手写的固定身份文本 |
| L1 Essential Story | ~500-800 tokens | 始终加载 | 从所有 drawers 按 importance/emotional_weight 排名，取 top-15，按 room 分组，hard cap 3200 chars |
| L2 On-Demand | ~200-500 tokens/次 | 对话触及某个 wing/room 时 | `col.get()` 带 where 过滤，不做向量搜索，按元数据过滤 |
| L3 Deep Search | 无限制 | 主动搜索时 | `col.query()` 语义搜索，返回相似度 + verbatim 原文 |

关键实现细节：

**L1 排分策略**（`layers.py:126-137`）：优先读取 `importance`，fallback 到 `emotional_weight`，再 fallback 到 `weight`，都没有则默认为 3。排完后按 room 分组，每条 snippet 截断为 200 chars，总量超 3200 chars 时输出截断提示 `"... (more in L3 search)"`。

**批量分页读取**（`layers.py:99-119`）：ChromaDB 有 SQLite 变量数量上限（约 999），L1 使用 500-条批次 + offset 循环读取全部 drawers，避免 `SQLite variable limit` 错误。

**MemoryStack 统一接口**：`wake_up()` = L0 + L1，`recall(wing, room)` = L2，`search(query)` = L3。调用方不需要知道底层分层细节。

### 2. searcher.py — 语义搜索

两个接口，职责分离：

- `search()` — 直接打印，给 CLI 用，输出带边框的格式化文本，包含 wing/room/source/similarity
- `search_memories()` — 返回 dict，给 MCP server 和程序调用，返回 `{query, filters, results: [{text, wing, room, source_file, similarity}]}`

搜索流程：`chromadb.PersistentClient` → `col.get_collection("mempalace_drawers")` → `col.query(query_texts=[query], n_results=n, include=["documents","metadatas","distances"])` → 相似度 = `1 - distance`（ChromaDB L2 距离转余弦近似）。

**Metadata 过滤**（`searcher.py:36-51`）：wing + room 双条件用 `{"$and": [{"wing": wing}, {"room": room}]}` 合并，单条件直接 `{"wing": wing}` 或 `{"room": room}`，不过滤时不传 `where`。过滤在向量搜索前由 ChromaDB 完成，是纯 metadata 精确匹配而非语义匹配。

### 3. convo_miner.py — 对话 ingest 和分块策略

对话文件的 ingest 核心是**交换对分块（exchange pair chunking）**：一次用户 turn（`>` 开头行）+ 紧随的 AI 响应 = 一个 drawer。

**格式探测**（`convo_miner.py:57-65`）：统计文件中 `>` 开头行数，≥3 条则认为是对话格式走 `_chunk_by_exchange()`，否则 fallback 到段落分块。

**`_chunk_by_exchange()` 细节**（`convo_miner.py:68-101`）：AI 响应最多取 8 行（`ai_lines[:8]`），join 成单行。chunk 最小 30 chars（`MIN_CHUNK_SIZE`），小于阈值丢弃。这意味着长 AI 响应被硬截断——原文的后半部分不进入该 drawer。

**Room 自动检测**（`convo_miner.py:129-206`）：对话内容有 5 个预定义 room 类别（technical/architecture/planning/decisions/problems），每类 10-13 个关键词，统计每类词频，取最高分。默认 room 为 `"general"`。

**两种 extract_mode**：
- `"exchange"`（默认）：Q+A 配对分块，保留上下文连贯性
- `"general"`：调用 `general_extractor.py`，提取 5 类记忆（decisions/preferences/milestones/problems/emotional），room 由 memory_type 决定

**幂等保护**（`convo_miner.py:223-228`）：`file_already_mined()` 在 `collection.get(where={"source_file": source_file}, limit=1)` 判断，已处理的文件跳过。

### 4. knowledge_graph.py — 时态知识图谱

独立于向量库，用 SQLite 存储结构化实体关系，与 Zep 的时态 KG 竞争但完全本地免费。

**数据模型**（`knowledge_graph.py:57-86`）：
- `entities` 表：id / name / type / properties(JSON) / created_at
- `triples` 表：subject / predicate / object / valid_from / valid_to / confidence / source_closet / source_file

**时态查询**（`knowledge_graph.py:188-243`）：`query_entity(name, as_of=None, direction="outgoing")` 通过 SQL `AND (valid_from IS NULL OR valid_from <= ?) AND (valid_to IS NULL OR valid_to >= ?)` 实现时间点过滤——`as_of="2026-01-15"` 只返回 2026 年 1 月 15 日时仍然有效的事实。

**事实失效**：`invalidate(subject, predicate, obj, ended)` 执行 `UPDATE triples SET valid_to=?` 将关系标记为已结束，而不删除记录，保留完整历史。

**自动实体创建**（`knowledge_graph.py:136-138`）：`add_triple()` 时，subject 和 object 不存在则自动 `INSERT OR IGNORE INTO entities`，无需先手动创建实体节点。

**WAL 模式**（`knowledge_graph.py:91-93`）：每次连接后执行 `PRAGMA journal_mode=WAL`，允许读写并发，避免 ChromaDB + SQLite 混合使用时的锁竞争。

**KG 与向量库桥接**：`source_closet` 字段存 drawer ID，可以从 KG 的事实回溯到 ChromaDB 中的 verbatim 原文，但这个桥接是可选的，实际 MCP 工具中 `tool_kg_add` 接受 `source_closet` 但不强制关联。

### 5. palace_graph.py — Room 导航图

从 ChromaDB metadata 动态构建 room 图，无需单独的图数据库。

**图构建逻辑**（`palace_graph.py:33-96`）：扫描所有 drawers 的 metadata，以 `room` 为节点，统计每个 room 出现在哪些 wings 中。如果一个 room 跨越多个 wing，则认为它是"tunnel"——连接不同项目/领域的共同话题（如 `chromadb-setup` 同时出现在 wing_code 和 wing_myproject）。

**BFS 遍历**（`palace_graph.py:99-158`）：`traverse(start_room, max_hops=2)` 从起始 room 出发，BFS 遍历通过共享 wing 连接的其他 room，返回按（hop 距离, count 倒序）排序的结果，最多返回 50 条。

**Fuzzy match**（`palace_graph.py:216-227`）：room 不存在时给出模糊建议，基于子字符串匹配，不是向量相似度。

**实用场景**：`find_tunnels(wing_a="wing_code", wing_b="wing_team")` 找出同时出现在代码工作和团队协作两个维度中的话题，揭示跨域关联。

### 6. dialect.py — AAAK 压缩方言

AAAK 是 mempalace 特有的**有损摘要格式**，明确不是无损压缩，原始文本无法从 AAAK 重建。

**格式结构**（`dialect.py:16-41`）：
```
Header:  FILE_NUM|PRIMARY_ENTITY|DATE|TITLE
Zettel:  ZID:ENTITIES|topic_keywords|"key_quote"|WEIGHT|EMOTIONS|FLAGS
Tunnel:  T:ZID<->ZID|label
Arc:     ARC:emotion->emotion->emotion
```

**实体编码**（`dialect.py:373-385`）：优先查预配置 entity_codes 字典，未命中则取名字前 3 个字符大写（`name[:3].upper()`）。

**`compress()` 流程**（`dialect.py:545-590`）：
1. `_detect_entities_in_text()` — 找已知实体或首字母大写词
2. `_extract_topics()` — 词频统计，跳过停用词，booost 大写词和含 `-`/`_` 的技术词
3. `_extract_key_sentence()` — 按决策词（decided/because/chose 等）评分，选最高分句子，截断为 52 chars
4. `_detect_emotions()` — 关键词信号匹配 20 种情感代码
5. `_detect_flags()` — 关键词信号匹配 7 种 flag（DECISION/ORIGIN/CORE/SENSITIVE/PIVOT/GENESIS/TECHNICAL）

**重要：AAAK 是"closets"层**，对应向量库中的"drawers"（verbatim 原文）。README 宣传的 96.6% 基准分数来自 raw mode（直接向量搜索原文），**不是** AAAK 压缩模式。AAAK 模式的基准退回至 84.2%。

### 7. mcp_server.py — MCP 工具完整列表

共 19 个工具（README 说 19 个，实际代码中 TOOLS dict 有 19 条目）：

**Read 工具（7 个）**：
- `mempalace_status` — 总览 + 向 AI 返回 PALACE_PROTOCOL 和 AAAK_SPEC（隐式教学）
- `mempalace_list_wings` — 所有 wings 及 drawer 数量
- `mempalace_list_rooms` — 指定 wing 的 rooms
- `mempalace_get_taxonomy` — 完整 wing → room → count 树
- `mempalace_get_aaak_spec` — 返回 AAAK 方言规范
- `mempalace_search` — 语义搜索，可选 wing/room 过滤
- `mempalace_check_duplicate` — 按相似度阈值（默认 0.9）检查重复

**Write 工具（2 个）**：
- `mempalace_add_drawer` — 存 verbatim 内容，写入前自动去重检查
- `mempalace_delete_drawer` — 按 ID 删除 drawer

**Knowledge Graph 工具（5 个）**：
- `mempalace_kg_query` — 查实体关系（支持 as_of 时间点过滤）
- `mempalace_kg_add` — 添加关系三元组
- `mempalace_kg_invalidate` — 标记关系失效
- `mempalace_kg_timeline` — 实体时间线
- `mempalace_kg_stats` — KG 统计

**Graph 工具（3 个）**：
- `mempalace_traverse` — BFS 从 room 出发遍历连接
- `mempalace_find_tunnels` — 找跨 wing 连接的 rooms
- `mempalace_graph_stats` — 图结构统计

**Agent Diary 工具（2 个）**：
- `mempalace_diary_write` — AI 写私人日记（AAAK 格式），按 agent_name 隔离 wing
- `mempalace_diary_read` — 读历史日记，按 `filed_at` 降序排列

**PALACE_PROTOCOL 隐式注入**（`mcp_server.py:92-99`）：`tool_status()` 的返回值中嵌入了 5 条行为规范（先查再说、不确定就查、每次会话写日记等），通过 wake-up 调用自动传递给 AI，无需 system prompt 工程。

### 8. onboarding.py — 初始化流程

6 步引导初始化，创建结构化 "world model"：

1. **Mode 选择**：work / personal / combo，决定默认 wing 分类（`DEFAULT_WINGS` 字典）
2. **People 录入**：姓名 + 关系 + 昵称，区分 personal/work context
3. **Projects 录入**：防止项目名被误识别为人名
4. **Wings 确认**：展示推荐 wings，允许自定义
5. **Auto-detect**：扫描目录用 `entity_detector.py` + `entity_registry.py` 发现更多名字，置信度 ≥ 0.7 才显示候选
6. **Ambiguity 警告**：名字和常用英文单词冲突时（`COMMON_ENGLISH_WORDS` 集合）告知用户

**产出两个文件**：
- `~/.mempalace/aaak_entities.md` — AAAK 实体注册表（人名 → 3 字符代码）
- `~/.mempalace/critical_facts.md` — Bootstrap 事实（people/projects/wings），会在后续被 palace_facts.py 增补

### 9. config.py — 配置系统

**三级优先级**（`config.py:65-98`）：环境变量（`MEMPALACE_PALACE_PATH` 或 `MEMPAL_PALACE_PATH`）> `~/.mempalace/config.json` > 硬编码默认值。

**关键默认值**：
- `palace_path`: `~/.mempalace/palace`（ChromaDB 持久化目录）
- `collection_name`: `"mempalace_drawers"`
- `topic_wings`: 7 个默认维度（emotions/consciousness/memory/technical/identity/family/creative）
- `hall_keywords`: 7 个主题 hall 的关键词列表，用于 miner 自动路由

**people_map 独立文件**（`config.py:106-114`）：人名映射（别名 → 正式名）存在 `~/.mempalace/people_map.json` 中，`config.json` 也可以内嵌，两者合并。

### 10. normalize.py — 格式归一化

支持 6 种来源格式，全部归一为 `> user_turn\nassistant_response\n\n` 格式：

| 格式 | 识别方式 | 处理逻辑 |
|------|---------|---------|
| 已有 `>` 标记的文本 | 统计 `>` 行 ≥ 3 | 直接透传 |
| Claude Code JSONL | `type` 字段为 `"human"/"user"/"assistant"` | 提取 message.content |
| OpenAI Codex CLI JSONL | `type="event_msg"` + `session_meta` 元数据 | 仅提取 `event_msg` 类型（跳过 response_item 防重复） |
| Claude.ai JSON | `messages` 或 `chat_messages` 字段，可处理 privacy export（嵌套 convo 数组）| role 字段识别 |
| ChatGPT conversations.json | `mapping` 树状结构 | 从根节点 BFS 遍历，按 children 链走最长路径 |
| Slack JSON export | `type="message"` 列表，多人交替发言 | 前两个不同 user 映射为 user/assistant 交替 |
| 普通文本 | fallback | 透传，按段落或行组分块 |

**Spellcheck hook**（`normalize.py:284-293`）：`_messages_to_transcript()` 可选调用 `mempalace.spellcheck.spellcheck_user_text`，纠正用户输入拼写错误后再生成 transcript。import 失败时 graceful fallback，不影响主流程。

---

## Claude Code Hooks 集成

`hooks/` 目录提供两个 shell 脚本，通过 Claude Code 的钩子机制自动触发：

- **Stop hook** (`mempal_save_hook.sh`)：每隔 15 条消息（可配置）在 session 结束时保存新信息
- **PreCompact hook** (`mempal_precompact_hook.sh`)：context window 即将压缩前保存当前上下文

配置方法（`examples/HOOKS_TUTORIAL.md`）：在 Claude Code 的 settings.json 中添加 `hooks.Stop` 和 `hooks.PreCompact` 配置，事件触发时自动调用对应脚本。这是被动式记忆积累，用户无需主动调用。

---

## 设计亮点

**Raw verbatim 作为核心哲学**：mempalace 是唯一把"不做 LLM 提取"作为设计核心的系统，不是能力不足而是主动选择。`miner.py` 文件顶部注释写明 `"No summaries. Ever."` 这个哲学有实证支撑：LongMemEval R@5 96.6% 超过 Mem0（~85% with LLM）、Mastra（94.87% with LLM）。信息论角度：不抽取 = 不丢失，有损压缩永远比无损检索的信息量少。

**PALACE_PROTOCOL 自注入**：`tool_status()` 的返回 JSON 中直接内嵌 PALACE_PROTOCOL 字符串（`mcp_server.py:92-99`），AI 调用 wake-up 时自动学会协议规范，无需手工系统提示工程。这是一种"在数据里教模型"的轻量内省机制。

**时态知识图谱本地化**：`knowledge_graph.py` 用 SQLite + WAL 模式实现 Zep 需要 Neo4j 云服务才能做的时态三元组图。`as_of` 参数支持时间点查询（"这个关系在 2026 年 1 月是否有效"），`invalidate()` 保留历史而不删除，完全本地，零成本。

**多层渐进加载**：4 层设计控制了 wake-up token 预算——L0+L1 仅 600-900 tokens，保留 95% 以上的上下文窗口给实际对话。相比全量注入（Hermes Agent 内置层）或无加载（纯搜索），这是token经济与检索覆盖的平衡点。

**Agent Diary 作为跨 session 自我记录**：`tool_diary_write` 让每个 AI 有独立 wing 存自己的日记（AAAK 格式），`tool_diary_read` 跨 session 读取历史。这是"AI 有持久自我意识"的架构实现，而不只是"用户数据的检索工具"。

**格式归一化广度**：`normalize.py` 支持 6 种来源格式（含 ChatGPT mapping 树、OpenAI Codex JSONL、Claude.ai privacy export），对话格式的处理用 `_chunk_by_exchange()` 把 Q+A 作为语义单元而不是硬按 token 切割，保留了对话上下文的完整性。

---

## 局限性

**AAAK 有损压缩退步明显**：AAAK 模式 LongMemEval 84.2% vs raw mode 96.6%，退步 12+ 个百分点。`compress()` 的 `_extract_key_sentence()` 截断为 52 chars，`_extract_topics()` 只取 top-3 词，信息损失严重。AAAK 适合当"方向指针"（closet），不适合作为实际检索的记忆内容。

**Room 自动检测粗粒度**：`detect_convo_room()` 只有 5 个预定义类别（technical/architecture/planning/decisions/problems），keyword 词频匹配不考虑语境。大量不含这些关键词的对话一律归入 `"general"` room，导致 general 成为垃圾桶，削弱 L2 过滤的精度。

**L2 不做语义搜索**：`Layer2.retrieve()` 使用 `col.get()` 按 metadata 精确过滤，不执行语义搜索。这意味着 "recall(wing='work')" 返回 wing=work 的任意 drawers 而非与当前对话相关的 drawers，召回相关性完全依赖 room 分类精度。

**KG 与向量库脱节**：`knowledge_graph.py` 的 triples 和 ChromaDB 的 drawers 是两个独立数据存储，`source_closet` 字段是可选的软连接，没有强制一致性保证。KG 中的事实和向量库中的原文可能描述的是同一个事情但互相不知道，需要 AI 手动维护两者的关联（通过 MCP 工具）。

**存储无增长上限**：raw verbatim 的代价是存储量随对话量线性增长。没有归档、剪枝、或自动过期机制（不像 Claude Code Auto-Dream 的 4 阶段 Prune 流程）。长期使用后 `~/.mempalace/palace/` 的 ChromaDB 体积可能显著膨胀，L1 的 MAX_DRAWERS=15 截断会越来越漏掉重要历史。

**`_chunk_by_exchange` 的 AI 响应截断**：`_chunk_by_exchange()` 只保留 AI 响应的前 8 行（`ai_lines[:8]`），长响应被硬截断。这破坏了 raw verbatim 的核心承诺——如果 AI 在第 9 行才给出关键答案，该信息被永久丢弃。

**无注入防护**：相比 Hermes Agent 的 `_scan_memory_content()`（11 类威胁模式扫描），mempalace 的 `tool_add_drawer` 没有任何写入前校验。任何内容都直接进入向量库，prompt injection 攻击可以通过 `mempalace_add_drawer` 植入恶意记忆，下次 wake-up 时污染 L1。

---

## 与其他源的横向对比

| 维度 | mempalace | Claude Code | DeerFlow | Hermes Agent |
|------|----------|-------------|----------|-------------|
| 存储策略 | raw verbatim，永不摘要 | LLM 后台提取 → Markdown | LLM 结构化 → JSON | LLM 写入 → MEMORY.md |
| 检索方式 | ChromaDB 向量搜索 | 词法匹配 | 置信度排序 | 全量注入（内置）+ 外部向量 |
| 信息损失 | 无（存原文） | 有（LLM 摘要可能漏） | 有（提取失真） | 有（LLM 筛选） |
| 结构化层 | KG（可选，SQLite） | 无 | JSON schema | 无（内置层） |
| 空间隐喻 | Wing/Hall/Room/Drawer | 无 | 无 | 无 |
| 跨 session 自我记录 | Agent Diary（MCP） | Auto-Dream | 无 | Nudge review |
| 安全防护 | 无 | 无 | 无 | 11 类威胁扫描 |
| 维护机制 | 无自动剪枝 | Auto-Dream（Prune 阶段） | 置信度降级 | Nudge review |

---

## 关键代码路径

- `mempalace/layers.py:76-177` — `Layer1.generate()`，L1 批量读取 + importance 排分 + 分组输出
- `mempalace/layers.py:369-448` — `MemoryStack`，4 层统一接口
- `mempalace/searcher.py:93-152` — `search_memories()`，MCP 语义搜索
- `mempalace/convo_miner.py:54-101` — `chunk_exchanges()` + `_chunk_by_exchange()`，Q+A 配对分块
- `mempalace/convo_miner.py:129-206` — `detect_convo_room()`，5 类 room 自动分类
- `mempalace/knowledge_graph.py:55-86` — SQLite schema，triples + valid_from/valid_to 时态字段
- `mempalace/knowledge_graph.py:188-243` — `query_entity()`，时态查询 SQL
- `mempalace/knowledge_graph.py:171-184` — `invalidate()`，事实失效而不删除
- `mempalace/palace_graph.py:33-96` — `build_graph()`，从 ChromaDB metadata 动态构建图
- `mempalace/palace_graph.py:99-158` — `traverse()`，BFS 房间遍历
- `mempalace/dialect.py:545-590` — `compress()`，AAAK 有损摘要流水线
- `mempalace/mcp_server.py:92-118` — `PALACE_PROTOCOL` + `AAAK_SPEC` 嵌入 status 响应
- `mempalace/mcp_server.py:349-392` — `tool_diary_write()`，agent 日记
- `mempalace/mcp_server.py:442-689` — `TOOLS` dict，19 个工具完整定义
- `mempalace/normalize.py:23-49` — `normalize()`，格式探测主入口
- `mempalace/onboarding.py:266-315` — `_generate_aaak_bootstrap()`，生成 AAAK 实体注册表

---

## 来源

- 源码路径：`/Users/neo/Desktop/project/git/mempalace/`
- 分析深度：源码级（layers.py / searcher.py / convo_miner.py / knowledge_graph.py / palace_graph.py / dialect.py / mcp_server.py / onboarding.py / config.py / normalize.py / general_extractor.py / miner.py 全部核心模块）
- benchmark 数据来源：`benchmarks/BENCHMARKS.md`（March 2026 数据）
