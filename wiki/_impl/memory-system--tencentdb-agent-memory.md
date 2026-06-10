---
title: "Memory System — TencentDB Agent Memory"
category: L2
parent: "[[memory-system]]"
source: tencentdb-agent-memory
source_version: "v0.3.6-8-gf92b102"
confidence: high
created: 2026-06-09
updated: 2026-06-09
relations:
  - target: "[[memory-system]]"
    type: implements
  - target: "[[context-management]]"
    type: feeds
  - target: "[[tool-system]]"
    type: depends_on
  - target: "[[symbolic-context-map-progressive-recall]]"
    type: supports
  - target: "[[tencentdb-agent-memory--mmd-symbolic-context-map]]"
    type: contains
---

## 概述

TencentDB Agent Memory 是一个**本地优先、可服务化扩展的分层记忆系统**。它有两条互补链路：

1. **短期 context offload**：把大工具结果落到本地 `refs/*.md`，用 `offload-<sessionId>.jsonl` 保存 L1 摘要，再用 L2 Mermaid/MMD 符号图维护当前任务状态，最后用 L3 根据上下文压力替换或删除消息。
2. **长期 memory**：L0 原始对话、L1 结构化记忆、L2 scene blocks、L3 persona 四层存储与召回。默认后端是本地 SQLite；FTS5 是本地检索路径，sqlite-vec 只有在配置真实 embedding 维度后才实际建表，也可以切到 Tencent Cloud VectorDB。

它最值得借鉴的不是单个数据库后端，而是**高密度导航层**：让 LLM 先看到压缩后的符号地图和场景索引，只有信息不足时再主动调用 memory/conversation search 或 `read_file` 展开细节。

---

## 架构概览

```text
短期 offload 链路:

tool call/result
  │
  ├─ L1.1 write refs/*.md                  ← 原始工具结果真值源
  │
  ├─ L1 summarize → offload-<session>.jsonl ← tool_call / summary / result_ref / score
  │
  ├─ L1.5 task judgment                     ← 判断当前任务是否长任务、是否延续旧 MMD
  │
  ├─ L2 MMD generation/update               ← Mermaid 认知状态机 + node_mapping
  │
  └─ L3 compression                         ← mild replace / aggressive delete / emergency

长期 memory 链路:

completed turn
  │
  ├─ L0 conversations/YYYY-MM-DD.jsonl      ← 原始对话消息
  ├─ L1 records/YYYY-MM-DD.jsonl + vectors  ← 结构化记忆、FTS/vector 索引
  ├─ L2 scene_blocks/*.md + scene index      ← 场景画像与事件块
  └─ L3 persona.md                           ← 用户画像 / 稳定偏好
```

默认本地目录分两套：

- 短期 offload：`~/.openclaw/context-offload/<agent>/`
- 长期 memory：`~/.openclaw/memory-tdai/`

---

## 短期 Context Offload

### 1. 本地文件结构

`src/offload/storage.ts` 定义了 immutable `StorageContext`，每个 agent/session 都有明确路径：

| 路径 | 职责 |
|---|---|
| `dataRoot` | 默认 `~/.openclaw/context-offload` |
| `dataDir` | 每个 agent 一个子目录；多 worker 场景会把 worker suffix 合入 agentName |
| `refs/` | 保存完整工具结果 Markdown |
| `mmds/` | 保存 Mermaid/MMD 任务状态图 |
| `offload-<sessionId>.jsonl` | 当前 session 的 offload 摘要行 |
| `state.json` | active MMD、cursor、counter 等插件状态 |
| `sessions-registry.json` | `sessionKey -> realSessionId` 映射 |

关键代码：

- `src/offload/storage.ts:17-54` — `DEFAULT_DATA_ROOT` 与 `StorageContext`
- `src/offload/storage.ts:73-90` — sessionKey 解析与 worker 隔离
- `src/offload/storage.ts:104-160` — session registry
- `src/offload/types.ts:12-30` — `OffloadEntry` schema，包含 `summary`、`result_ref`、`tool_call_id`、`node_id`、`score`

### 2. L1/L1.5/L2/L3 职责

**L1：工具结果摘要 + 原文引用**

`src/offload/index.ts` 在处理 pending tool pairs 时先写 ref 文件，再调用 L1 生成 summary。即使 LLM summary 失败，也会写 degraded fallback entry，确保链路不断。

- `src/offload/index.ts:434-445` — L1.1 将完整工具结果写入 `refs/*.md`
- `src/offload/index.ts:466-486` — L1 调用摘要并补齐 `result_ref`
- `src/offload/index.ts:505-515` — L1 degraded fallback

**L1.5：任务边界判断**

L1.5 判断当前用户轮次是长任务、短任务、延续历史任务，决定是否创建/复用 active MMD。它还读取已有 MMD metadata，支持任务切换时把旧任务残留 null entries flush 到旧 MMD，避免误归入新任务。

- `src/offload/index.ts:536-647` — L1.5 判断、activeMmd 切换、残留 flush

**L2：MMD 符号图生成**

L2 prompt 明确要求用 Mermaid flowchart 表达任务拓扑，并要求每个新 `tool_call_id` 都映射到 `node_mapping` 中某个节点。节点可以一对多聚合多个工具调用，图内节点带 status、summary、timestamp，目标是给 LLM 一个高密度状态地图。

- `src/offload/local-llm/prompts/l2-prompt.ts:9-30` — 高密度语义图、节点合并、符号即语义
- `src/offload/local-llm/prompts/l2-prompt.ts:36-54` — JSON 输出结构，含 `node_mapping`
- `src/offload/local-llm/prompts/l2-prompt.ts:118-125` — 将新 offload entries 交给 L2 合并入图
- `src/offload/pipelines/l2-mermaid.ts:96-218`、`src/offload/index.ts:689-781` — 按 MMD 分批 L2 update、回填 node ids

注意：`node_mapping` 的全覆盖是 **prompt 约束**，不是代码硬校验。解析器只读取已有 mapping；缺失 mapping 时，`backfillNodeIds` 会使用最频繁 node 或 MMD 文本中的最新 node 作为 fallback，仍无可用节点时跳过。这个实现保证链路不断，但也意味着 MMD 归因质量需要额外观测。

**L3：上下文压力下的消息替换/删除**

L3 在 `after_tool_call` / `before_prompt_build` 路径中根据窗口占用做 mild、aggressive、emergency 三档处理：

- mild：优先把非当前任务的大工具结果替换为 summary/ref
- aggressive：删除更旧消息，并用 history MMD 作为上下文替代
- emergency：极端情况下更强力删除到目标水位

可配置阈值主要在 `src/config.ts:208-268`，包括 `mildOffloadRatio`、`aggressiveCompressRatio`、`mmdMaxTokenRatio`；`emergencyCompressRatio` / `emergencyTargetRatio` 属于 offload runtime defaults，在 `src/offload/types.ts:181-187` 定义并由运行时读取。

预算控制有一个实现细节：history MMD 注入会按 `contextWindow * mmdMaxTokenRatio` 做 full/meta/skip 降级；active MMD 注入函数会计算和 trace budget，但不会在注入点硬截断 active MMD，实际增长主要依赖 L2 prompt 的 4000 字约束和节点合并策略。

### 3. MMD 注入给 LLM

MMD 不是后台内部结构，它会被注入到消息流给主 LLM 作为导航。

- `src/offload/mmd-injector.ts:24-40` — 每轮 user message / LLM call 可注入 active MMD
- `src/offload/mmd-injector.ts:77-85` — active MMD 作为 `role: user` 文本消息插入
- `src/offload/hooks/after-tool-call.ts:214-222` — 注入 `<current_task_context>`，说明 done/doing 节点含义

插入点有工具调用配对保护：`findActiveMmdInsertionPoint()` 不会把 MMD 插到 assistant tool_use 和 tool_result 中间。

- `src/offload/mmd-injector.ts:184-199` — 插入策略说明
- `src/offload/mmd-injector.ts:228-260` — 向前调整，避免拆散 tool pair

---

## 长期 Memory

### 1. Store 抽象与后端

长期存储由 `IMemoryStore` 统一抽象，上层 hooks/tools/pipeline 只依赖接口，不依赖具体后端。

- `src/core/store/types.ts:1-16` — backend-agnostic / capability-based / fault-tolerant 设计原则
- `src/core/store/types.ts:177-190` — `StoreCapabilities`：vector/FTS/native hybrid/sparse vector
- `src/core/store/types.ts:219-312` — `IMemoryStore`：L0/L1 写入、查询、搜索、profile sync、reindex

默认后端是本地 SQLite：

- `src/core/store/factory.ts:5-8` — `sqlite` 默认，本地 SQLite + sqlite-vec + FTS5；`tcvdb` 为 Tencent Cloud VectorDB
- `src/core/store/factory.ts:90-125` — sqlite 分支创建 `vectors.db`
- `src/core/store/sqlite.ts:1-21` — SQLite 同时管理 L1/L0 relational metadata + vec0 virtual table

但零配置不等于立即拥有向量检索：

- `src/config.ts:360-370` — 默认 `provider="none"`，embedding 被禁用
- `src/config.ts:421-427` — `provider="none"` 时 dimensions 为 0
- `src/core/store/sqlite.ts:590-601`、`src/core/store/sqlite.ts:849-850` — 只有 `dimensions > 0` 才创建 vec0 表并标记 ready

因此更准确的表述是：默认本地 SQLite 可用；FTS5 best-effort；sqlite-vec 是可启用能力，不是零配置即生效的能力。

切到 Tencent Cloud VectorDB 时必须配置 `tcvdb.url`、`apiKey`、`database`：

- `src/config.ts:287-290` — `storeBackend: "sqlite" | "tcvdb"`
- `src/core/store/factory.ts:50-88` — tcvdb 分支校验配置并创建 `TcvdbMemoryStore`

### 2. 四层长期记忆

| 层 | 内容 | 自动注入/按需展开 |
|---|---|---|
| L0 raw conversation | 原始 user/assistant 消息，daily JSONL + L0 vector/FTS index | 不默认全文注入；通过 `tdai_conversation_search` 按需查 |
| L1 structured memory | LLM 从 L0 提取的结构化记忆，含 type/priority/scene/time | 每轮 auto-recall 检索少量相关记忆注入；也可 `tdai_memory_search` |
| L2 scene blocks | 场景文件 `scene_blocks/*.md` 与 scene navigation | scene navigation 稳定注入；完整 scene 由 `read_file` 按需读 |
| L3 persona | `persona.md` 用户画像/长期偏好 | 稳定注入 system 末尾 |

关键代码：

- `src/core/conversation/l0-recorder.ts:1-15` — L0 daily JSONL 原始对话记录
- `src/core/hooks/auto-capture.ts:145-156` — L0 vector indexing：SQLite 可先写 metadata/FTS，后台补 embedding；远程后端同步写
- `src/core/record/l1-extractor.ts:1-13` — L1 从 L0 提取 scene-segmented structured memories
- `src/core/scene/scene-extractor.ts:1-17` — L2 scene extractor 用带工具的 LLM agent 读写 scene block
- `src/core/persona/persona-generator.ts:1-4` — L3 persona generator

### 3. Auto Recall 注入策略

`performAutoRecall()` 在每轮 LLM 处理前运行：

- L1 相关记忆放入 `prependContext`，作为动态 user prompt 前缀；
- L2 scene navigation、L3 persona、memory tools guide 放入 `appendSystemContext`，作为相对稳定、可 prompt-cache 的 system 后缀；
- 若注入片段不足，guide 告诉 LLM 可调用 memory/conversation search 或 read_file 继续展开。

关键代码：

- `src/core/hooks/auto-recall.ts:31-48` — memory tools guide，并限制 memory search 每轮合计最多 3 次
- `src/core/hooks/auto-recall.ts:115-141` — L1 search + budget 裁剪
- `src/core/hooks/auto-recall.ts:144-170` — L3 persona + L2 scene navigation
- `src/core/hooks/auto-recall.ts:186-218` — stable/dynamic 分离，照顾 prompt caching

注意：每轮最多 3 次 memory search 当前是 guide/tool description 约束，注册点还标注 hard per-turn limit 待实现；`performAutoRecall()` 自身不是 function call，而是在 hook 内直接查询 store。也就是说，它会增加“可被模型主动调用的 recall 工具面”，但自动召回本身不消耗模型 tool call。

### 4. Agent 可调用搜索工具

两个核心 recall 工具：

- `tdai_memory_search`：查 L1 结构化记忆，适合偏好、规则、历史事件节点。
- `tdai_conversation_search`：查 L0 原始对话，适合查具体原文、时间线、上下文细节。

搜索策略均支持 hybrid：FTS5 keyword + vector embedding 并行，RRF 融合；缺少某一路时自动降级。

- `src/core/tools/memory-search.ts:1-10` — L1 search 工具说明
- `src/core/tools/memory-search.ts:118-180` — FTS/vector 并行搜索
- `src/core/tools/conversation-search.ts:1-10` — L0 search 工具说明
- `src/core/tools/conversation-search.ts:117-180` — FTS/vector 并行搜索
- `src/core/tdai-core.ts:286-326` — `searchMemories()` / `searchConversations()` 映射到工具

实现上还有一个值得区分的点：auto-recall 在 TCVDB `nativeHybridSearch` 可用时会走单次 native hybrid shortcut；两个 agent-callable search tools 固定按 FTS/vector 两路并行再 RRF merge，没有同样的 native hybrid shortcut。对 agent-os 来说，这说明“后台自动召回”和“模型主动搜索工具”可以共用 store 抽象，但仍应分别做延迟/成本优化。

---

## 设计亮点

### 1. 符号图作为高密度 context map

MMD 把大量工具调用压成节点、状态、边和简短 summary，既保留任务拓扑，又避免把原始工具结果长期塞进上下文。它比普通摘要更可导航：LLM 能看到当前 doing 节点、已完成节点、blocked 雷区和文件名。

### 2. 原文真值源与 LLM 可见摘要分离

完整工具结果在 `refs/*.md`，上下文里只放 summary/MMD/ref。这样既能降 token，又保留可追溯证据。对 `agent-os` 来说，这比单纯 `ToolResultBudget` 超限 nudge 更强：后续可按 ref 召回，而不是只能要求用户/模型重跑工具。

### 3. 自动注入 + 主动召回组合

L1/L2/L3 给 LLM 足够多的导航信息；当不足时，LLM 再通过 memory search / conversation search / read_file 展开。这是“默认低 token，必要时多一次 tool call”的 progressive disclosure。

### 4. 本地优先但有服务化出口

短期 offload 默认写本地文件，长期 memory 默认 SQLite；但配置支持 backend mode、TCVDB、Gateway。它不是只为云产品设计，也不是只能单机运行。

### 5. Store 能力探测和降级

`StoreCapabilities` 把 vector/FTS/hybrid/sparse vector 能力显式化，搜索工具能按能力降级。SQLite 路径还支持 metadata/FTS 先写、embedding 后台补齐，避免 agent_end 被 embedding 阻塞。

---

## 局限与风险

### 1. 会增加少量 function calling

如果注入的 MMD/L1 snippets 不足，LLM 可能会多调用 `tdai_memory_search`、`tdai_conversation_search` 或 `read_file`。Tencent 在 guide 和工具描述中要求每轮 memory search 合计最多 3 次，但源码注册点仍标注 hard limit 待实现；`read_file` 场景也需要宿主工具侧的预算控制。

### 2. MMD 质量依赖 L2 LLM

L2 prompt 要求“每个 tool_call_id 都映射到 node_mapping”，但图的节点拆合、状态和摘要仍由 LLM 判断。若 L2 归纳错误，主 LLM 会被错误地图误导；需要观测 node_mapping 覆盖率和 MMD 变更质量。

### 3. 注入为 user-role 文本有语义风险

MMD 和 `<current_task_context>` 被作为 user-role 消息插入。虽然标签说明“仅供参考”，但它不是 provider 原生 system/developer 层；在更严格的 context protocol 中，应考虑用专门的 runtime/context section 呈现，避免被误认为用户新指令。

### 4. 文件系统并发和清理需要运营策略

短期 offload 的 refs/jsonl/mmd/state 都在本地文件系统，适合 local profile。Web 分布式 profile 需要替换成 object store + DB index，并处理多 worker 并发写、清理、TTL 和权限隔离。

### 5. MMD token 预算仍会占上下文

MMD 本身也会增长。系统通过 4000 字 prompt 约束、history MMD 的 `mmdMaxTokenRatio` 降级、history MMD 只在 aggressive 压缩后注入等方式控制；但 active MMD 注入没有硬截断，长期多任务仍需要归档/冷热分层，否则符号图也会成为新的 token 占用源。

---

## 对 agent-os 的借鉴点

1. 新增 `ToolResultEvidenceStore`：大工具结果先落 evidence store，消息里写 summary + ref + token metadata。
2. 新增 `TaskContextMap` / `SymbolicContextMap`：从工具结果、状态更新和任务事件生成高密度任务地图。
3. 扩展 `recall_context`：支持按 segment、node、evidence ref 三种粒度召回。
4. 区分 local/distributed profile：local 用 filesystem/SQLite；distributed 用 object store/Postgres/Redis/vector DB。
5. 给 recall 工具设 per-turn budget：例如 memory/conversation/evidence recall 合计最多 3 次，避免搜索循环。

## 来源

- 项目：`/Users/neo/Desktop/project/git/TencentDB-Agent-Memory`
- 源码版本：`v0.3.6-8-gf92b102`
- 分析深度：源码级（offload、core store、auto-recall、tools、scene/persona pipeline）
