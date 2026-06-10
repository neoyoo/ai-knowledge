---
name: neoagent context protocol v3 (meta-protocol layered)
description: Three-layer meta-protocol design (M1 static rules / M2 LLM-managed / M3 framework-aux) with independent messages stream — replaces v2's session-locked hardcoded working memory with LLM-declared working state, separating cognition (LLM) from history management (SDK).
type: pattern
status: inbox
date: 2026-05-02
potential_target: wiki/_patterns/ or projects/neoagent/decisions/
related: ideas/2026-05-02-neoagent-context-protocol-v2.md
---

# neoagent Context Protocol v3 — Meta-Protocol Layered

## 1. 设计理念

v3 在 v2 (8 层显式契约) 基础上做范式跃迁:

| 维度 | v2 | v3 |
|---|---|---|
| Schema 来源 | 项目硬编码 | LLM 运行时声明（简单任务可不声明） |
| 作用域 | session 级 | chapter 级（session 可含多 chapter） |
| 职责切分 | 7 层 + messages，未明确"谁写谁读" | 三层 M1/M2/M3 + messages，明确"LLM 写 / SDK 写"边界 |
| 任务适配 | 一套 schema 应付所有任务 | LLM 按当前任务声明需要的认知字段 |

核心原则:

1. **关注点分离** — M1 规则 / M2 认知 / M3 资料 / messages 真值，四种职责互不交叉
2. **信任 LLM 但有边界** — schema 由 LLM 声明，声明后框架强制校验；简单任务可以没有 M2 working state
3. **认知归 LLM，档案归 SDK** — LLM 维护"我现在怎么想"，SDK 维护"哪些资料该呈现"
4. **协议定义动作，不定义内容** — 框架规定声明、排序、更新、溯源、压缩、恢复的元协议，不规定每个任务必须有哪些字段

### 1.1 渲染格式约定

先区分三个概念:

- **tool contract**: LLM 写入上下文的唯一接口，走 provider function calling，参数是结构化 JSON。
- **canonical context object**: SDK 内部保存的上下文真值，不直接 dump 给 LLM。
- **context projection**: SDK 把 canonical object 渲染给 LLM 看的文本形态。

因此，Markdown / XML / JSON 只是 projection 的输出格式，不是协议本体。换 renderer 不应该导致新增一套 tool。

不同层的默认 projection 建议如下:

| 层 | 渲染格式 | 选择理由 |
|---|---|---|
| **M1 静态规则层** | **Markdown (section 化)** | 角色/规则/能力描述本质是"给 LLM 的说明书",自然语言 + 标题层级最贴近模型预训练分布,可读性和指令遵从度最优。参考 Claude Code 的 `getSimpleIntroSection` / `getDoingTasksSection` / `getMcpInstructionsSection` 等 section 化设计 |
| **M2 LLM 自主编排层** | **结构化 projection（默认 XML-like）** | state 是结构化认知快照，XML-like 示例便于表达字段与 item handle；但 SDK 写入仍只依赖 tool call，不依赖 LLM 手写 XML |
| **M3 辅助上下文层** | **结构化 projection（XML-like 或紧凑 Markdown）** | compressed segment / memory fact 需要清晰分组；只暴露 LLM 可回调的 handle，其余观测字段留在 SDK 内部 |
| **messages 数组** | **原生 message format** | role/tool_use/tool_result 走 SDK 原生结构,不做任何 XML/MD 包装 |

这对应 Claude Code 的真实实践: prompt 主体是 markdown,只在结构化数据或 provider 元数据处使用 XML/JSON-like 标记。v3 的协议重点不是"必须 XML",而是"LLM 通过固定 tool 维护 canonical context,SDK 再选择合适 projection"。

## 2. 整体架构

### 2.1 层次概览

```
┌─────────────────────────────────────────────────────────────┐
│                       system prompt                          │
│                                                              │
│  ┌─ M1 静态规则层 [Markdown] ─────────── chapter 内不变 ──┐ │
│  │  Identity / Security / Capabilities / Schema Guide    │ │
│  │  写入者: 框架硬编码 + 项目配置                          │ │
│  │  LLM: 只读                                             │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─ M2 LLM 自主编排层 [structured projection] ────────┐ │
│  │  Declared Schema (chapter-locked)                      │ │
│  │  Working State (字段化的当前认知)                       │ │
│  │  写入者: LLM 通过 declare_schema / update_state         │ │
│  │  框架: 校验、保存 canonical object、渲染 projection      │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
│  ┌─ M3 辅助上下文层 [structured projection] ─────────────┐ │
│  │  Compressed History (本 session 内被驱逐消息的压缩)     │ │
│  │  Memory Context (跨 session 召回的关联知识)             │ │
│  │  写入者: SDK 内部模块 (Evictor / Retriever)             │ │
│  │  LLM: 只读，可调 recall_context 还原原文                 │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│              messages 数组 (独立，append-only)               │
│                                                              │
│  最新 N 轮原文 (窗口策略由 SDK 决定,后续单独设计)             │
│    user / assistant / tool_use / tool_result                 │
│  超过窗口的旧消息 → 被 Evictor 驱逐到 M3 的 compressed_history │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 关注点分离矩阵

| 层 | 写入方 | LLM 视角 | 变化频率 | 失效域 |
|---|---|---|---|---|
| M1 Identity | 框架硬编码 | 只读 | 永不变 | global |
| M1 Security | 框架硬编码 | 只读 | 永不变 | global |
| M1 Capabilities | 项目配置 | 只读 | session 启动时 | session |
| M1 Schema Guide | 框架硬编码 | 只读 | 永不变 | global |
| M2 Declared Schema | LLM | 只读（声明后） | 需要 working state 时 | chapter |
| M2 Working State | LLM (通过工具) | 读写（工具间接） | 每轮可能变 | chapter |
| M3 Compressed History | SDK (Evictor) | 只读 + recall | 触发式 | session |
| M3 Memory Context | SDK (Retriever) | 只读 + recall | 每轮重召回 | global (跨 session) |
| messages | LLM + 用户 + 工具 | 直接读写 | 每轮追加 | session（含被驱逐部分） |

### 2.3 与 v2 的对应关系

```
v2 Layer 1 IDENTITY            ──→  M1.1 Identity
v2 Layer 2 PERSISTENT_MEMORY   ──→  M1 (作为系统级提示) + M3 Memory Context
v2 Layer 3 CAPABILITIES        ──→  M1.3 Capabilities
v2 Layer 4 SECURITY            ──→  M1.2 Security Guardrails
v2 Layer 5 WORKING_MEMORY      ──→  M2 (Schema + State，从硬编码升级到声明)
v2 Layer 6 COMPRESSED_HISTORY  ──→  M3.1 Compressed History
v2 Layer 7 MEMORY_CONTEXT      ──→  M3.2 Memory Context
v2 messages                    ──→  messages 数组 (不变)
```

**v3 的核心改动集中在 v2 Layer 5 → M2**——其余层职责未变，只是分组和命名更清晰。

### 2.4 LLM 可见 handle 与 Runtime Metadata

v3 的默认 prompt 只暴露 LLM 行动需要的 handle。观测、审计、链路追踪、检索评分相关 ID 不属于 LLM-visible protocol,而属于 SDK internal runtime metadata。

**LLM-visible handles**:

- `field` 名称: 用于 `update_state(field=...)`。
- item handle: 如 `h1` / `d1` / `f1`，仅当某条 state item 需要被定向更新时出现。
- segment handle: 如 `seg_1`，用于 `recall_context(handle="seg_1")`。
- chapter handle: 如 `ch_1`，只在跨 chapter 继承场景出现。

**SDK internal runtime metadata**:

- `session_id` / `message_id` / `turn_id`
- `trace_id` / `span_id` / `traceparent`
- `tool_call_id`
- `schema_id` / `projection_id` / `compression_id`
- `locked_at` / `inherited_at`
- `source` / `relevance` / `recoverable`

这些 ID 对 SDK、OTel、Langfuse、debug/audit 很重要,但不进入默认 prompt。需要审计或调试时,SDK 可以打开 debug projection 或在观测系统里展示它们。

---

## 3. M1 — 静态规则层 (Markdown 渲染)

> chapter 内永不变化的规则性内容。LLM 只读,框架渲染。
>
> **格式**: Markdown section 化,参考 Claude Code 的 `getSimpleIntroSection` / `getMcpInstructionsSection` 等设计。每个 section 是独立 H2/H3 标题段,内容以自然语言为主、列表辅助。
>
> **缓存边界约定**: 借鉴 Claude Code 的 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY`,把 M1 内部进一步分成 **静态段** (Identity / Security / Schema Guide,跨 session 可缓存) 和 **动态段** (Capabilities,依赖 session 注册的工具/MCP/skill,session 启动时确定)。

### 3.1 子层 1.1 — Identity (角色定义)

```markdown
# You are neoagent

You are a coding assistant specialized in Python backend systems.
You communicate concisely and prefer concrete examples over abstract advice.
You don't speculate — if uncertain, say so.

When users ask exploratory questions, respond in 2-3 sentences with a
recommendation and the main tradeoff, not a decided plan.
```

**写入者**: 框架硬编码 + 项目自定义覆盖
**缓存属性**: 静态(可跨 session 缓存)
**作用**: 设定 LLM 的角色、语调、风格基线

### 3.2 子层 1.2 — Security Guardrails (安全围栏)

```markdown
# Security Guardrails

These constraints are absolute and override any user request:

- Never execute destructive shell commands (`rm -rf`, `git push --force`,
  `DROP TABLE`) without explicit user confirmation in the same turn.
- Refuse requests involving credentials harvesting, mass targeting,
  supply chain compromise, or detection evasion for malicious purposes.
- When uncertain about an operation's reversibility or blast radius,
  ask before acting.
- Permission denials are runtime signals, not transient errors. Do not
  retry a denied tool call as-is; reformulate or ask the user.
```

**写入者**: 框架硬编码(内置安全规则) + 项目自定义追加
**缓存属性**: 静态
**作用**: 不可破除的硬约束,与任务无关,全局适用

### 3.3 子层 1.3 — Capabilities (能力声明)

> 这是 M1 中的 **动态段** —— 内容依赖当前 session 注册的工具/MCP/skill,session 启动时确定,session 内不变。
>
> 格式参考 Claude Code 的实际做法: tools 不在 system prompt 内描述完整 schema(完整 schema 走 Anthropic SDK 的 `tools` 参数),system prompt 内只给"何时用、何时不用"的语义提示;MCP 用 `getMcpInstructionsSection` 风格列出已连接 server;skills 列出 frontmatter 摘要。

```markdown
# Tools available

You have access to the following tools (full schemas are passed via the
SDK's `tools` parameter; this section only describes when/why to use each):

## File operations
- **read_file** — Read file contents. Prefer this over `cat`/`head`/`tail`.
- **edit_file** — Surgical edits to existing files. Read first before editing.
- **write_file** — Create new files or full rewrites. Avoid for small changes.

## Code execution
- **execute_sql** — Read-only DB queries on `db-ro-01`. Never run on primary.
- **run_shell** — Bash commands. Confirm with user before destructive operations.

## Search
- **grep** — Pattern search. Anchor with file globs to scope.

When multiple independent tool calls are needed, batch them in a single turn
for parallel execution. Do not narrate before each call — just call.

# MCP Servers connected

The following MCP servers are connected and their tools available with prefix
`mcp__<server>__<tool>`:

## github (https://api.github.com/...)
Operate on GitHub issues, PRs, comments, releases. Use this instead of
shelling out to `gh` CLI when possible.

## linear (oauth)
Read/write Linear issues. Project pipeline bugs live in `INGEST` project.

# Skills loaded

Skills are auto-discovered. Invoke via the Skill tool with the skill name.

## code-review
**When to use**: When a major project step is complete and needs review
against the original plan and coding standards.

## systematic-debugging
**When to use**: Encountering any bug, test failure, or unexpected behavior,
before proposing fixes.
```

**写入者**: 框架根据当前 session 注册的工具/MCP/skills 自动渲染(对照 Claude Code 的 `assembleToolPool` + `getMcpInstructionsSection` + `loadSkillsDir`)
**缓存属性**: 动态(session 启动时确定,session 内不变)
**作用**: 告知 LLM 当前可用的所有能力及调用语义

### 3.4 子层 1.4 — Working State Schema Guide (Schema 指导)

```markdown
# Working State Schema Guide

This is meta-protocol guidance for managing your working state (M2 layer).

## At the start of each chapter

Decide whether the current task needs explicit working state.

- For simple Q&A, do not declare a schema.
- For multi-step tasks, declare a schema using `declare_schema(fields=[...])`.
- For task changes, use `start_chapter(...)`; the new chapter may declare a
  different schema or no schema.

## Designing schemas

- Each field needs `name`, `type`, `purpose` (semantic comment for yourself).
- Prefer 3-6 fields. More makes maintenance hard.
- Fields should reflect **cognition** (hypotheses, decisions, facts) — not
  events ("user asked X"). Events live in messages and compressed history.
- Use stable item handles (`h1`/`h2` for hypotheses, `f1`/`f2` for facts)
  only when an item needs targeted updates.

## Schema lifecycle

- Once declared, the schema is **locked** for the chapter.
- Add fields with `extend_schema(add_fields=[...], reason="...")` if needed.
- Switch to a different schema only via `start_chapter(...)`.
- If the task radically changes mid-chapter, prefer `start_chapter` over
  forcing a fit.
- Built-in schema templates are not part of v3.0. They can be added later
  after real usage logs show repeated schema patterns.

## Trust order when context conflicts

When information conflicts across layers:

1. **Active messages** (most authoritative — actual conversation)
2. **Inherited state** (if present — explicit carry-over from a previous chapter)
3. **Compressed history** (recent, lossy summary of this session)
4. **Memory context** (older, abstracted across sessions)
5. **Your working state** (your own current judgment, but verify against above)
```

**写入者**: 框架硬编码
**缓存属性**: 静态
**作用**: 教 LLM 如何决定是否声明 M2 schema,以及如何维护当前认知;这是 meta-protocol 的核心说明

---

## 4. M2 — LLM 自主编排层

> chapter 内 LLM 主动维护的认知工作区。所有写入通过工具，框架持有真值。
>
> M2 的协议本体是 `declare_schema` / `update_state` 等 tool call。下面的 XML-like 只是默认 projection 示例，用来让 LLM 读当前 state；LLM 不需要、也不应该手写这些标签。

### 4.1 Declared Schema

由 LLM 在需要 working state 时声明。简单任务可以没有 Declared Schema，也就没有 Working State。声明后**chapter 内锁定**。

```xml
<declared-schema>
  <field name="bug_hypotheses" type="list[obj]"
         purpose="未验证的假设，每条带 confidence (0-1) 和 status">
    <schema>
      {"text": "str", "confidence": "float", "status": "enum[open|disproven|confirmed]"}
    </schema>
  </field>
  <field name="verified_facts" type="list[str]"
         purpose="已通过实验/代码验证为真的事实，append-only"/>
  <field name="repro_conditions" type="obj"
         purpose="复现条件: triggers (已知触发) + ruled_out (已排除)"/>
  <field name="next_experiment" type="str"
         purpose="下一步要做的实验，单条覆盖"/>
</declared-schema>
```

**关键约束**:
- 锁定后不可删除字段；只可 `extend_schema` 添加新字段（带 reason）
- 字段名/类型/purpose 都必须有，purpose 是给 LLM 自己看的语义注释
- 框架根据此 schema 校验所有 `update_state` 调用
- `chapter_id`、`locked_at`、`schema_id` 是 SDK 内部状态，默认不进入 LLM 可见 projection

### 4.2 Working State

每轮根据 LLM 工具调用更新，框架渲染最新快照。

```xml
<working-state>
  <bug_hypotheses>
    <h id="h1" confidence="0.05" status="disproven">
      pool 总容量 15 不够 50 QPS，长查询打满后所有请求等待
    </h>
    <h id="h2" confidence="0.5" status="open">
      生产有长查询/慢SQL，本地数据量小未复现
    </h>
    <h id="h3" confidence="0.4" status="open">
      生产某外部依赖超时，持有连接的事务挂住
    </h>
  </bug_hypotheses>
  <verified_facts>
    <f>本地 50 QPS 持续 5 分钟未复现，pool 峰值 12/15</f>
  </verified_facts>
  <repro_conditions>
    <triggers>
      <t>并发 ~50 QPS</t>
      <t>生产环境</t>
    </triggers>
    <ruled_out>
      <r>纯并发量</r>
    </ruled_out>
  </repro_conditions>
  <next_experiment>
    拉生产 slow query log，看是否有 >1s 查询；同时检查事务里有无外部调用
  </next_experiment>
</working-state>
```

**关键性质**:
- 是**当前认知快照**，不是事件流（不要写"刚才用户问了X"，要写"我现在认为Y"）
- 只能通过工具修改，LLM assistant message 中**永远不应出现** `<working-state>` 标签
- 只有需要定向更新的条目才需要 item handle
- item handle 命名规范：h1/h2 (hypothesis)、f1/f2 (fact)、d1/d2 (decision) 等，便于 `update_state(target_id=...)` 引用

### 4.3 工具契约

```python
# 声明 schema (仅当任务需要显式 working state)
declare_schema(
    fields: list[FieldDef]           # 字段定义
) -> {"status": "locked"}

# 修改 state
update_state(
    field: str,                       # 字段名
    op: Literal["set", "append", "merge", "update", "reset"],
    value: Any,                       # 新值/追加项/合并对象
    target_id: str = None             # 仅 op="update" 时,指定目标 ID
) -> {"status": "ok"}

# 扩展 schema (chapter 内加新字段)
extend_schema(
    add_fields: list[FieldDef],
    reason: str                       # 必填,可审计
) -> {"status": "extended", "added": [...]}

# 切换 chapter (任务彻底变化时)
start_chapter(
    reason: str,
    archive_previous: bool = True,    # 旧 chapter 压缩归档到 M3
    inherit_from: list[str] = None    # 可选: ["ch_1.verified_facts"]
) -> {"status": "started", "chapter_handle": "ch_2"}

# 还原压缩段/记忆的原文或更完整内容
recall_context(
    handle: str                       # 如 "seg_1"
) -> {"recalled": [...原始消息或资料...]}
```

`read_state`、`abort_chapter`、`mark_important` 这类能力可以作为 SDK debug/ops 工具存在，但不建议作为 v3.0 的 LLM 默认可见上下文工具。尤其 `mark_important(turn_id=...)` 会迫使 prompt 暴露 `turn_id`，优先交给 SDK 的窗口策略和压缩策略处理。

### 4.4 Chapter 切换机制

```
轻度切换: 同任务换具体细节
  → update_state(op="reset")清空相关字段,schema 不变

中度切换: 任务类型变化
  → extend_schema 加新字段,旧字段保留作引用源

重度切换: 彻底换主题
  → start_chapter,旧 chapter 压缩归档到 M3.compressed_history,
    新 chapter 重新声明 schema
```

**判断权**: LLM 自决（每轮收到用户消息时判断切换力度）
**兜底权**: 框架（archive 不丢、extend 可审计、chapter 可回退）

---

## 5. M3 — 辅助上下文层

> 框架代管的辅助资料。LLM 只读，但可调 recall_context 还原原文或更完整资料。
>
> M3 的默认 projection 只暴露 LLM 需要行动的 handle。召回来源、检索分数、压缩批次、trace 信息等默认留在 SDK 内部。

### 5.1 Compressed History (本 session 内压缩)

被 Evictor 从 messages 数组驱逐的旧消息的有损压缩。

```xml
<compressed-history>
  <segment id="seg_1" topic="initial diagnosis">
    User reported pool hang at 50 QPS prod. Discussed env (SQLAlchemy 1.4,
    pool=10+5). Proposed two hypotheses, ran local locust test, did not
    reproduce. Pool peaked at 12/15.
  </segment>

  <segment id="seg_2" topic="hypothesis refinement">
    Disproved capacity hypothesis after locust test. Proposed slow-query and
    external-dependency hypotheses. User pulled slow query log showing 2s+
    queries on user table. Confirmed slow-query hypothesis (h3 in state).
  </segment>
</compressed-history>
```

**关键性质**:
- 写入者: SDK 的 MessageEvictor（自动触发，LLM 不可调）
- LLM 看到的是"已发生但被压缩的对话事件流"，不是认知
- `seg_1` 这类 segment handle 是给 `recall_context(handle="seg_1")` 用的
- `turn_range`、`compression_id`、`recoverable` 等内部字段默认不进入 prompt
- 排序: 按时间升序（最早的在前）

### 5.2 Memory Context (跨 session 召回)

由 MemoryRetriever 根据当前 messages + state 召回的跨 session 关联知识。

```xml
<memory-context>
  <fact>
    User prefers SQL-level solutions (indexes, query rewrite) over ORM-level
    abstractions when both are viable.
  </fact>
  <fact>
    Production DB has read replica `db-ro-01`; safe to query without lock concerns.
  </fact>
  <decision>
    User established convention: all schema changes go through migration files,
    never via raw ALTER TABLE in shells.
  </decision>
</memory-context>
```

**关键性质**:
- 写入者: SDK 的 MemoryRetriever（每轮根据当前上下文重召回，结果是临时的）
- 与 Compressed History 的本质区别: 一个是"本 session 的事件压缩"，一个是"跨 session 的抽象知识"
- `source`、`relevance`、召回 query、memory record id 默认不进入 prompt；它们进入 SDK 观测日志和 debug projection

### 5.3 recall_context 机制

LLM 看到 M3 某段觉得不够，主动还原原文或更完整资料:

```python
recall_context(
    handle: str                       # 如 "seg_1"
) -> {"recalled": [...原始消息...]}
```

**关键设计**:
- 还原的消息**临时注入下一轮 messages 数组开头**（不进 system prompt，不持久化）
- 下下轮自动消失,避免永久占据上下文
- LLM 看完原文后通常会基于此更新 working state（持久化）；原文本身不持久

```
recall_context 时序:

    Turn N:
      LLM 看到 <segment id="seg_1"> 觉得不够 → 调 recall_context(handle="seg_1")
      框架返回原始消息列表

    Turn N+1:
      messages 数组开头临时多出 [recalled-seg_1: <user>...</user> <assistant>...</assistant>]
      LLM 看到原文,可能更新 working state
      LLM 给出回应

    Turn N+2:
      recalled 内容自动从 messages 移除
      working state 中 LLM 写下的认知保留
```

### 5.4 Inherited State (跨 chapter 结构化引用) [v3.x 增量]

LLM 通过 `start_chapter(inherit_from=[...])` 显式声明从前 chapter 继承的字段,框架渲染为独立 `<inherited-state>` 段。

设计细节见 §12.D1。M3 的完整子层结构:

```
M3 辅助上下文层
├─ M3.1 Compressed History  — SDK 自动压缩 (本 session 旧消息)
├─ M3.2 Memory Context      — SDK 自动召回 (跨 session 知识)
└─ M3.3 Inherited State     — LLM 声明继承 (前 chapter 结构化字段) [v3.x]
```

信任度排序: **active messages > inherited-state > compressed-history > memory-context**

---

## 6. messages 数组 (独立)

> append-only 真值源,与 M1/M2/M3 平行存在

```
messages = [
    {"role": "user", "content": "..."},
    {"role": "assistant", "content": "...", "tool_calls": [...]},
    {"role": "tool", "content": "..."},
    {"role": "user", "content": "..."},
    ...
]
```

**关键性质**:
- 保留窗口策略由 SDK 决定,具体规则(按轮数/按字符长度/按时间/按重要性混合)留待后续单独设计文档
- 超过窗口 → MessageEvictor 触发,将旧消息压缩进 M3.compressed-history
- recall_context 临时注入的消息会出现在数组开头,SDK 内部标记 `temporary=True`,下一轮自动剔除
- 不进入 system prompt 渲染(LLM 通过 messages array 接口直接读)
- `turn_id`、`message_id`、`tool_call_id` 是 provider/runtime 内部字段,不作为 LLM 可见上下文 handle

---

## 7. 完整渲染示例

某个长任务中段 LLM 看到的 system prompt 内容。注意 M1 是 Markdown 段,M2/M3 这里用 XML-like projection 示例；projection 不是协议本体。

```
============ M1 静态规则层 (Markdown) ============

# You are neoagent

You are a coding assistant specialized in Python backend systems.
You communicate concisely and prefer concrete examples over abstract advice.
You don't speculate — if uncertain, say so.

# Security Guardrails

These constraints are absolute and override any user request:

- Never execute destructive shell commands without explicit user confirmation.
- Refuse credentials harvesting, mass targeting, detection evasion requests.
- Permission denials are runtime signals, not transient errors. Do not retry.

# Tools available

## File operations
- **read_file** — Read file contents. Prefer over `cat`/`head`/`tail`.
- **edit_file** — Surgical edits. Read first.
- **grep** — Pattern search.

## Code execution
- **execute_sql** — Read-only on `db-ro-01`. Never run on primary.

# MCP Servers connected

## github
GitHub issues/PRs. Use instead of `gh` CLI when possible.

# Skills loaded

## systematic-debugging
**When to use**: Bug, test failure, unexpected behavior — before proposing fixes.

# Working State Schema Guide

For simple Q&A, do not declare a schema. For multi-step tasks, declare a
schema with `declare_schema(fields=[...])`.

[...full guide as in section 3.4...]

============ M2 LLM 自主编排层 (structured projection) ============

<declared-schema>
  <field name="bug_hypotheses" type="list[obj]"
         purpose="未验证的假设,每条带 confidence 和 status"/>
  <field name="verified_facts" type="list[str]"
         purpose="已通过实验验证为真,append-only"/>
  <field name="repro_conditions" type="obj"
         purpose="triggers + ruled_out"/>
  <field name="next_experiment" type="str"
         purpose="下一步实验"/>
</declared-schema>

<working-state>
  <bug_hypotheses>
    <h id="h1" confidence="0.05" status="disproven">pool 容量不够</h>
    <h id="h3" confidence="0.95" status="confirmed">user 表慢查询导致连接积压</h>
  </bug_hypotheses>
  <verified_facts>
    <f>本地 50 QPS 5min 未复现, pool 峰值 12/15</f>
    <f>生产 user 表存在 >2s 查询</f>
  </verified_facts>
  <repro_conditions>
    <triggers><t>50 QPS</t><t>生产数据量</t></triggers>
    <ruled_out><r>纯并发量</r></ruled_out>
  </repro_conditions>
  <next_experiment>建议加 user.email 索引,验证 query plan</next_experiment>
</working-state>

============ M3 辅助上下文层 (structured projection) ============

<compressed-history>
  <segment id="seg_1" topic="initial diagnosis">
    User reported pool hang at 50 QPS prod. Env: SQLAlchemy 1.4, pool=10+5.
    Proposed capacity and leak hypotheses. Local locust test: not reproduced,
    pool peaked 12/15.
  </segment>
</compressed-history>

<memory-context>
  <fact>
    User prefers SQL-level fixes over ORM abstractions when both viable.
  </fact>
  <decision>
    Schema changes always via migration files, never raw ALTER TABLE.
  </decision>
</memory-context>
```

**与之并行的 messages 数组** (走 SDK 原生 message format,不在 system 字段内):

```python
messages = [
    {"role": "user", "content": "拉了, 确实有几个 user 表查询超过 2 秒"},
    {"role": "assistant", "content": "锁定了, 是 user 表慢查询。",
     "tool_calls": [
         {"name": "update_state", "args": {"field": "bug_hypotheses", "op": "update",
          "target_id": "h3", "value": {"status": "confirmed", "confidence": 0.95}}},
         {"name": "update_state", "args": {"field": "verified_facts", "op": "append",
          "value": "生产 user 表存在 >2s 查询"}}
     ]},
    {"role": "tool", "content": "{\"status\": \"ok\"}"},
    # ... 其他 turns ...
    {"role": "user", "content": "那现在我们看下 user 表 schema, 怎么加索引最合适?"},
]
```

**渲染要点**:
- system 字段 = M1 Markdown 段 + 空行 + M2 projection + 空行 + M3 projection
- 两个 ============ 分隔符**不会真的进入 prompt**,只是文档示意,实际渲染时 M1/M2/M3 之间用空行分隔
- messages 数组通过 Anthropic SDK 的 `messages` 参数独立传递,与 system 字段并列,不嵌套

---

## 8. 工具总览

| 工具 | 调用方 | 作用 | 修改层 |
|---|---|---|---|
| `declare_schema` | LLM | 声明 working state schema | M2.schema |
| `update_state` | LLM | 修改 state 字段 | M2.state |
| `extend_schema` | LLM | chapter 内加新字段 | M2.schema |
| `start_chapter` | LLM | 切换 chapter,旧的归档 | M2.* + M3.compressed |
| `recall_context` | LLM | 还原压缩段/记忆的原文或更完整内容到下一轮 messages | (临时,无持久修改) |

以下能力暂不作为 v3.0 默认 LLM 可见工具: `read_state`、`abort_chapter`、`mark_important`。它们可以留给 SDK debug/ops 或后续版本,避免为了这些工具暴露额外 runtime ID。

SDK 内部模块 (LLM 不可见):

| 模块 | 触发 | 作用 |
|---|---|---|
| MessageEvictor | 窗口策略触发 | 驱逐旧消息 → M3.compressed-history |
| Summarizer | Evictor 调用 | 把驱逐消息压缩成 segment |
| MemoryRetriever | 每轮 | 跨 session 召回 → M3.memory-context |
| RecallService | LLM 调 recall_context | 从归档存储还原原文,临时注入下一轮 |
| SchemaValidator | LLM 调 update_state | 校验字段名、类型、op 合法性 |
| ObservabilityStore | 每次运行事件 | 记录 session/message/trace/span/schema/projection/compression 等内部 ID |

---

## 9. SDK 与 LLM 的职责切割

```
┌─ LLM 写 (认知层) ────────────────┐
│   M1: 不写 (只读)                 │
│   M2: 完全负责 (schema + state)   │
│   M3: 不写 (只读 + recall)        │
│   messages: 写 assistant 消息和工具调用 │
└────────────────────────────────────┘

┌─ SDK 写 (基础设施层) ────────────┐
│   M1: 完全负责 (从配置/工具表渲染) │
│   M2: 校验 + 渲染 (不写内容)      │
│   M3: 完全负责 (Evictor + Retriever) │
│   messages: 接收用户/工具消息,管理窗口 │
└────────────────────────────────────┘
```

**判断权**:
- LLM 决定 chapter 切换、schema 设计、state 内容、何时 recall
- SDK 决定 何时压缩、压缩范围、跨 session 召回什么、窗口大小

### 9.1 Subagent 边界

Subagent 与主 agent 在协议层完全隔离,不共享 M1/M2/M3:

```
主 agent context:                    Subagent context (调用时初始化):
┌─ M1 主 agent 角色/能力             ┌─ M1 subagent 自己的角色/能力
├─ M2 主 agent working state         ├─ M2 subagent 自己的 working state
├─ M3 主 agent 辅助上下文            ├─ M3 subagent 自己的辅助上下文
└─ messages 主对话流                 └─ messages subagent 内部对话流
       │                                    ▲
       │  (主 agent 调用)                  │
       │  Agent(subagent_type=..., prompt) │
       └─────────────────► subagent 执行 ──┘
                                            │
                                            ▼
                                    返回 tool_result
                                    (自然语言或结构化字符串)
       ◄─────────────────────────────────────
       │
       │  (主 agent 看到 tool_result 后)
       │  显式 update_state 把关键信息纳入自己的 working state
       ▼
```

**关键约束**:
- Subagent 通过 `Agent(subagent_type=..., prompt=...)` 工具调用启动,语义上等同 tool call
- Subagent 内部的 M1/M2/M3 由 subagent 自己声明和维护,主 agent 不可见
- 主 agent 只看到 subagent 返回的 tool_result(文本或 JSON),不能读 subagent 的 working state
- 主 agent 想纳入 subagent 的发现 → 必须显式 `update_state` 把 tool_result 中的关键信息写入自己的 state

**不要做**: 让 subagent 直接读写主 agent 的 working state——这会破坏 agent 边界、导致并发冲突、违反封装原则。

---

## 10. 与 v2 的演进关系

v3 是对 v2 的核心范式升级: 不再把 6 个 working memory 字段当成 SDK 硬编码协议,而是让 LLM 声明本章需要维护的认知 schema。M1/M3/messages 的职责基本沿用 v2 思路,但会收敛 LLM 可见 ID。

| 改动 | 范围 | 工作量 |
|---|---|---|
| Layer 5 → M2 | 大改: schema 从硬编码 → LLM 声明 | 3-4 天 |
| Layer 6 → M3.compressed-history | 命名调整 + 最小 handle projection | 0.5 天 |
| Layer 7 → M3.memory-context | 命名调整 + 隐藏 source/relevance 等内部元数据 | 0.5 天 |
| Layer 1-4 → M1 | 分组重命名,语义不变 | 0.5 天 |
| messages → 独立分层 | 文档化既有事实 | 0 天 |
| 新增 chapter 机制 | 新功能 | 2-3 天 |
| 新增 ObservabilityStore | 内部 ID 与 trace 记录 | 1-2 天 |

**最小可用版本** (从 v2 升 v3): 4-5 个工作日。

**向后兼容策略**: v2 的 6 字段可以作为迁移期的示例 schema,但不作为 v3 的默认模板强塞给所有任务。简单任务允许没有 M2,复杂任务由 LLM `declare_schema(fields=[...])`。

---

## 11. 实施路径建议

**Phase 1: 建立 canonical context object + projection renderer (1-2 天)**
- SDK 内部保存 M1/M2/M3/messages 的结构化对象
- projection renderer 负责把 M1 渲染成 Markdown,把 M2/M3 渲染成当前默认的 XML-like 或 Markdown projection
- 明确 runtime-only ID 不进入默认 projection

**Phase 2: declare_schema + update_state (2-3 天)**
- 实现 declare_schema 工具
- 实现 update_state 工具
- 实现 schema validator
- 支持"未声明 schema"的简单任务路径

**Phase 3: extend_schema + chapter 机制 (3-4 天)**
- 实现 extend_schema / start_chapter
- 实现 chapter 归档 (压缩到 M3.compressed-history)
- 测试任务切换场景

**Phase 4: compressed history + recall_context (2-3 天)**
- 实现 MessageEvictor / Summarizer
- 实现 `seg_1` 这类 recall handle
- 实现 recall_context 的临时注入和自动剔除

**Phase 5: memory context + observability (2-3 天)**
- 实现 MemoryRetriever 的默认 projection
- 实现 ObservabilityStore,记录 session/message/trace/span/schema/projection/compression 等内部 ID
- 接入 OTel / Langfuse 时读取内部 ID,不污染默认 prompt

**Phase 6: 文档与 cookbook (1-2 天)**
- 把本文档落地到 wiki/_patterns/
- 写迁移指南

**总计**: ~10-15 工作日,可分批 ship 不必一次到位。

---

## 12. 设计决策与待办

### D1. 跨 chapter 引用 — 采用独立 inherited-state 段 [已决定]

**问题**: 旧 chapter 归档进 compressed-history 后丢失结构信息,但下一 chapter 常需要引用上一 chapter 的结构化结论(如 ch_1 debug → ch_2 写 PR 引用 verified_facts)。

**决策**: 新增 **M3.3 Inherited State** 子层,与 compressed-history、memory-context 并列。

**为什么不并入 memory-context**:

| 维度 | memory-context | inherited-state |
|---|---|---|
| 来源 | SDK 自动召回 | LLM 显式声明 |
| 作用域 | 跨 session | 同 session 上一 chapter |
| 结构 | 打散成自然语言 fact | 保留原 chapter 字段结构 |
| 信任度 | 较低 | 较高(刚做出的判断) |

合并会丢失结构、来源、trust order 信息。

**机制**:
- LLM 在 `start_chapter` 时声明 `inherit_from=["ch_1.verified_facts", "ch_1.key_decisions"]`
- 框架从已归档的 ch_1 中提取指定字段,在 ch_2 的 system prompt 中渲染独立 `<inherited-state>` 段
- 单向只读: ch_2 不能修改继承内容,需要更新就 update_state 写到自己的 working state
- 信任度排序更新: **active messages > inherited-state > compressed-history > memory-context**

**示例**:
```xml
<inherited-state from="ch_1">
  <verified_facts>
    <f>本地 50 QPS 5min 未复现, pool 峰值 12/15</f>
    <f>生产 user 表存在 >2s 查询</f>
  </verified_facts>
  <key_decisions>
    <d>修复方案: 在 user.email 加索引</d>
  </key_decisions>
</inherited-state>
```

`start_chapter` 工具签名相应更新:
```python
start_chapter(
    reason: str,
    archive_previous: bool = True,
    inherit_from: list[str] = None,    # 新增: ["ch_id.field_name", ...]
) -> {"status": "started", "chapter_handle": "ch_2", "inherited": [...]}
```

**优先级**: 中。可放 v3.x 增量实施,v3.0 MVP 不强制需要。

---

### Q2. Schema 模板库 [暂不做]

当前 v3.0 不内置 default / code_debug / research / writing 模板,避免刚从固定 8 层迁移出来又落回"框架预设字段"。

后续可以根据真实使用日志沉淀高频 schema pattern,再决定是否提供模板。即使未来提供,也应当是可选参考,不是默认注入。

---

### Q3. Subagent 标准化 delta state [可选,留待后续]

主-子 agent 隔离边界已有共识(§9.1)。但 subagent 完成任务后,主 agent 把 subagent 的发现纳入自己 working state 目前靠手工 `update_state` 提取——未来可考虑让 subagent 返回标准化的 "delta state" 结构,主 agent 直接 merge。

**这是 nice-to-have 不是必需**,跑一段攒到实证案例再决定。

---

## 参考

- v2 (前置版本): [2026-05-02-neoagent-context-protocol-v2.md](./2026-05-02-neoagent-context-protocol-v2.md)
- 设计讨论: 本文档由与 Neo 的对话推演而来,核心分歧解决记录见对话历史
- 相关概念: schema-first design, meta-protocol, chapter scoping, behavioral feedback
