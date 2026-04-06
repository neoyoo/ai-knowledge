# Ontology Scaffold Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Scaffold the knowledge base directory structure, schema definitions, page templates, and 12 initial L1 concept stubs, ready for content ingestion.

**Architecture:** Flat wiki structure with L1 concept pages at top level, L2 implementation pages in `_impl/`, and schema config in `schema/`. All files are markdown or YAML — no code runtime.

**Tech Stack:** Markdown (Obsidian-compatible), YAML, Git

**Spec:** `docs/superpowers/specs/2026-04-06-ontology-design.md`

---

## File Map

### Create:
- `raw/.gitkeep` — placeholder for raw sources directory
- `wiki/_index.md` — global entry point / concept map
- `wiki/_impl/.gitkeep` — placeholder for L2 implementation pages
- `schema/ontology.yaml` — concept definitions + relation types
- `schema/page-templates/L1.md` — L1 page template
- `schema/page-templates/L2.md` — L2 page template
- `schema/lint-rules.yaml` — basic linter rules
- `wiki/prompt-system.md` — L1 stub
- `wiki/query-loop.md` — L1 stub
- `wiki/tool-system.md` — L1 stub
- `wiki/context-management.md` — L1 stub
- `wiki/memory-system.md` — L1 stub
- `wiki/runtime-state.md` — L1 stub
- `wiki/multi-agent.md` — L1 stub
- `wiki/hooks.md` — L1 stub
- `wiki/mcp-skills.md` — L1 stub
- `wiki/session-recovery.md` — L1 stub
- `wiki/channel-remote.md` — L1 stub
- `wiki/evaluation-observability.md` — L1 stub

---

### Task 1: Create directory structure

**Files:**
- Create: `raw/.gitkeep`
- Create: `wiki/_impl/.gitkeep`
- Create: `schema/page-templates/.gitkeep`

- [ ] **Step 1: Create all directories with placeholders**

```bash
mkdir -p raw wiki/_impl schema/page-templates
touch raw/.gitkeep wiki/_impl/.gitkeep
```

- [ ] **Step 2: Verify structure**

```bash
find . -type d | grep -E '^\./(raw|wiki|schema)' | sort
```

Expected:
```
./raw
./schema
./schema/page-templates
./wiki
./wiki/_impl
```

- [ ] **Step 3: Commit**

```bash
git add raw/ wiki/ schema/
git commit -m "scaffold: create directory structure (raw, wiki, schema)"
```

---

### Task 2: Create ontology.yaml

**Files:**
- Create: `schema/ontology.yaml`

- [ ] **Step 1: Write ontology definition**

```yaml
# AI Knowledge Base — Ontology Definition
# This file is the source of truth for concepts and relation types.

version: "1.0"
updated: "2026-04-06"

# --- L1 Concepts ---
concepts:
  - id: prompt-system
    title: Prompt System
    aliases: [prompt 系统, prompt composition, dynamic prompting]
    definition: 动态组装发给模型的 prompt

  - id: query-loop
    title: Query Loop
    aliases: [agent loop, 主循环, agentic loop]
    definition: Agent 主循环 — 发请求、拿结果、判断下一步

  - id: tool-system
    title: Tool System
    aliases: [工具系统, tool dispatch, function calling, tool use]
    definition: 发现、选择、调用、管理外部工具

  - id: context-management
    title: Context Management
    aliases: [上下文管理, context window, token budgeting]
    definition: 管理有限的 context window

  - id: memory-system
    title: Memory System
    aliases: [记忆系统, persistent memory, cross-session memory]
    definition: 跨会话的持久记忆

  - id: runtime-state
    title: Runtime State
    aliases: [运行时状态, state management, agent state]
    definition: 运行时状态容器与生命周期

  - id: multi-agent
    title: Multi-Agent
    aliases: [多智能体, multi-agent orchestration, task delegation]
    definition: 多 agent 协作 — 拆任务、分发、汇总

  - id: hooks
    title: Hooks
    aliases: [钩子, event hooks, lifecycle hooks]
    definition: 事件驱动的扩展点

  - id: mcp-skills
    title: MCP & Skills
    aliases: [MCP, skills, 扩展协议, extension protocol]
    definition: 外部扩展协议 + 可复用能力包

  - id: session-recovery
    title: Session Recovery
    aliases: [会话恢复, checkpoint, fault tolerance]
    definition: 断点恢复与容错

  - id: channel-remote
    title: Channel & Remote
    aliases: [渠道, remote execution, multi-channel]
    definition: 多渠道接入与远程执行

  - id: evaluation-observability
    title: Evaluation & Observability
    aliases: [评估, observability, metrics, tracing]
    definition: 效果评估 + 运行可观测

# --- Relation Types ---
relations:
  - id: uses
    label: A 用了 B
    description: 运行时调用
    example: Query Loop → Tool System

  - id: feeds
    label: A 给 B 提供数据
    description: 数据流向
    example: Memory → Context Management

  - id: contradicts
    label: A 和 B 思路相反
    description: 理念冲突
    example: 长 context vs RAG

  - id: evolved-into
    label: A 变成了 B
    description: 技术演化
    example: Function Calling → Tool Use

  - id: alternative
    label: A 和 B 是替代方案
    description: 同一问题不同解法
    example: ReAct vs Plan-and-Execute

  - id: extends
    label: A 给 B 加了新能力
    description: 扩展
    example: MCP 扩展 Tool System

# --- Extension Rules ---
extension:
  conditions:
    - 至少 3 个不同项目/来源对该概念有独立实现
    - 不能被已有 L1 概念覆盖（否则归为 L2）
    - 需要 Neo 和 Claude 讨论审核确认
  exception: 设计足够优秀/创新的，可免除 3 来源要求，单源也可直接提为 L1
```

- [ ] **Step 2: Validate YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('schema/ontology.yaml')); print('YAML valid')"
```

Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add schema/ontology.yaml
git commit -m "schema: add ontology definition (12 concepts, 6 relation types)"
```

---

### Task 3: Create page templates

**Files:**
- Create: `schema/page-templates/L1.md`
- Create: `schema/page-templates/L2.md`

- [ ] **Step 1: Write L1 template**

Write to `schema/page-templates/L1.md`:

```markdown
---
title: "{{TITLE}}"
aliases: [{{ALIASES}}]
category: L1
created: {{DATE}}
updated: {{DATE}}
relations:
  - target: "[[related-concept]]"
    type: uses | feeds | contradicts | evolved-into | alternative | extends
sources:
  - source-project-name
---

## 一句话定义

{{DEFINITION}}

## 核心问题

这个组件要解决什么？
- 问题 1
- 问题 2

## 各家对比

| 维度 | 项目 A | 项目 B |
|------|--------|--------|
| 维度 1 | ... | ... |

## 设计权衡

- **选择 A vs 选择 B**：优劣分析...

## L2 详情

- [[concept--project-a]]
- [[concept--project-b]]
```

- [ ] **Step 2: Write L2 template**

Write to `schema/page-templates/L2.md`:

```markdown
---
title: "{{CONCEPT}} — {{SOURCE}}"
category: L2
parent: "[[{{PARENT}}]]"
source: {{SOURCE_ID}}
source_version: "{{VERSION}}"
confidence: high | medium | low
created: {{DATE}}
updated: {{DATE}}
---

## 概述

一段话概括该项目对这个概念的实现方式。

## 架构分析

### 子话题 1
具体实现分析...

### 关键代码路径
- `path/to/file` — 说明

## 设计亮点

- 亮点 1

## 局限性

- 局限 1

## 来源

- 源码版本：{{VERSION}}
- 分析深度：源码级 | 文档级 | 博客级
```

- [ ] **Step 3: Commit**

```bash
git add schema/page-templates/
git commit -m "schema: add L1 and L2 page templates"
```

---

### Task 4: Create _index.md global entry

**Files:**
- Create: `wiki/_index.md`

- [ ] **Step 1: Write index page**

Write to `wiki/_index.md`:

```markdown
---
title: AI Agent Knowledge Base
category: index
updated: 2026-04-06
---

# AI Agent Knowledge Base

聚焦智能体开发的结构化知识库。以组件为维度，跨项目对比设计。

## 概念地图

### Core Runtime
- [[prompt-system]] — 动态组装 prompt
- [[query-loop]] — Agent 主循环
- [[runtime-state]] — 运行时状态与生命周期
- [[context-management]] — Context window 管理

### Capabilities
- [[tool-system]] — 工具发现/调用/管理
- [[memory-system]] — 跨会话持久记忆
- [[multi-agent]] — 多 agent 协作

### Extension
- [[hooks]] — 事件驱动扩展点
- [[mcp-skills]] — 外部扩展协议 + 能力包
- [[channel-remote]] — 多渠道接入

### Reliability
- [[session-recovery]] — 断点恢复与容错
- [[evaluation-observability]] — 效果评估 + 运行可观测

## 来源项目

| 项目 | 状态 | L2 页数 |
|------|------|---------|
| Claude Code | 已分析 | 0 (待导入) |
| OpenAI Agents SDK | 计划中 | — |
| Dify | 计划中 | — |
| LangGraph | 计划中 | — |

## 关系类型

| 关系 | 含义 |
|------|------|
| A 用了 B | 运行时调用 |
| A 给 B 提供数据 | 数据流向 |
| A 和 B 思路相反 | 理念冲突 |
| A 变成了 B | 技术演化 |
| A 和 B 是替代方案 | 同一问题不同解法 |
| A 给 B 加了新能力 | 扩展 |
```

- [ ] **Step 2: Commit**

```bash
git add wiki/_index.md
git commit -m "wiki: add global index page with concept map"
```

---

### Task 5: Create 12 L1 concept stubs (batch 1 — Core Runtime)

**Files:**
- Create: `wiki/prompt-system.md`
- Create: `wiki/query-loop.md`
- Create: `wiki/runtime-state.md`
- Create: `wiki/context-management.md`

- [ ] **Step 1: Write prompt-system.md**

```markdown
---
title: Prompt System
aliases: [prompt 系统, prompt composition, dynamic prompting]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[context-management]]"
    type: uses
sources: []
---

## 一句话定义

Agent 运行时中负责动态组装发给模型的 prompt 的子系统，不是静态模板。

## 核心问题

- 怎么把系统指令、用户输入、工具结果、历史对话组装成一个 prompt？
- 怎么根据运行时状态动态调整 prompt 内容？
- 怎么在 token 限制内取舍？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 2: Write query-loop.md**

```markdown
---
title: Query Loop
aliases: [agent loop, 主循环, agentic loop]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[tool-system]]"
    type: uses
  - target: "[[prompt-system]]"
    type: uses
  - target: "[[context-management]]"
    type: uses
sources: []
---

## 一句话定义

Agent 的主循环 — 发请求给模型、拿结果、判断下一步（继续/调工具/结束），再来一轮。

## 核心问题

- 循环什么时候结束？谁来判断？
- 一轮里面的状态怎么传递？
- 出错了怎么处理（重试/降级/终止）？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 3: Write runtime-state.md**

```markdown
---
title: Runtime State
aliases: [运行时状态, state management, agent state]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[session-recovery]]"
    type: feeds
sources: []
---

## 一句话定义

Agent 运行时的状态容器 — 管理当前会话信息、配置、运行模式和生命周期。

## 核心问题

- 哪些状态是全局的，哪些是单轮的？
- 状态怎么序列化（给 checkpoint/recovery 用）？
- 多 agent 场景下状态怎么隔离？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 4: Write context-management.md**

```markdown
---
title: Context Management
aliases: [上下文管理, context window, token budgeting]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[prompt-system]]"
    type: feeds
  - target: "[[memory-system]]"
    type: uses
sources: []
---

## 一句话定义

管理有限的 context window — 什么放进去、什么压缩、什么丢掉。

## 核心问题

- 当对话超长时，怎么决定保留哪些信息？
- 压缩策略：截断 vs 摘要 vs 向量检索？
- Token 预算怎么分配给不同模块（系统指令/历史/工具结果）？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 5: Verify all 4 files have valid frontmatter**

```bash
for f in wiki/prompt-system.md wiki/query-loop.md wiki/runtime-state.md wiki/context-management.md; do
  echo "--- $f ---"
  python3 -c "
import yaml
with open('$f') as fh:
    content = fh.read()
    fm = content.split('---')[1]
    data = yaml.safe_load(fm)
    print(f'  title: {data[\"title\"]}, category: {data[\"category\"]}, relations: {len(data.get(\"relations\", []))}')
"
done
```

- [ ] **Step 6: Commit**

```bash
git add wiki/prompt-system.md wiki/query-loop.md wiki/runtime-state.md wiki/context-management.md
git commit -m "wiki: add L1 stubs — Core Runtime (prompt, loop, state, context)"
```

---

### Task 6: Create 12 L1 concept stubs (batch 2 — Capabilities)

**Files:**
- Create: `wiki/tool-system.md`
- Create: `wiki/memory-system.md`
- Create: `wiki/multi-agent.md`

- [ ] **Step 1: Write tool-system.md**

```markdown
---
title: Tool System
aliases: [工具系统, tool dispatch, function calling, tool use]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[mcp-skills]]"
    type: extends
  - target: "[[query-loop]]"
    type: feeds
sources: []
---

## 一句话定义

Agent 运行时中负责发现、选择、调用和管理外部工具的子系统。

## 核心问题

- 怎么让模型知道有哪些工具可用？
- 怎么把模型的意图转成实际的工具调用？
- 怎么处理权限和安全？
- 工具调用失败了怎么处理？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 2: Write memory-system.md**

```markdown
---
title: Memory System
aliases: [记忆系统, persistent memory, cross-session memory]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[context-management]]"
    type: feeds
  - target: "[[runtime-state]]"
    type: uses
sources: []
---

## 一句话定义

跨会话的持久记忆 — 和 context 不同，活过对话结束，下次还在。

## 核心问题

- 什么值得记住，什么不值得？
- 记忆怎么存储（文件/数据库/向量）？
- 怎么在新对话开始时检索相关记忆？
- 记忆过时了怎么处理？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 3: Write multi-agent.md**

```markdown
---
title: Multi-Agent
aliases: [多智能体, multi-agent orchestration, task delegation]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[runtime-state]]"
    type: uses
sources: []
---

## 一句话定义

多个 agent 协作 — 怎么拆任务、怎么分发、怎么汇总结果。

## 核心问题

- 什么时候该拆成多个 agent，什么时候单个就够？
- Agent 之间怎么通信？共享状态还是消息传递？
- 子 agent 失败了怎么处理？
- 怎么避免重复工作？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 4: Commit**

```bash
git add wiki/tool-system.md wiki/memory-system.md wiki/multi-agent.md
git commit -m "wiki: add L1 stubs — Capabilities (tool, memory, multi-agent)"
```

---

### Task 7: Create 12 L1 concept stubs (batch 3 — Extension)

**Files:**
- Create: `wiki/hooks.md`
- Create: `wiki/mcp-skills.md`
- Create: `wiki/channel-remote.md`

- [ ] **Step 1: Write hooks.md**

```markdown
---
title: Hooks
aliases: [钩子, event hooks, lifecycle hooks]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: extends
sources: []
---

## 一句话定义

事件驱动的扩展点 — 在 agent 运行的特定时机插入自定义逻辑。

## 核心问题

- 哪些生命周期事件值得暴露为 hook？
- Hook 执行失败会阻塞主流程吗？
- 怎么保证 hook 的执行顺序？
- 用户怎么注册和管理 hook？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 2: Write mcp-skills.md**

```markdown
---
title: MCP & Skills
aliases: [MCP, skills, 扩展协议, extension protocol]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[tool-system]]"
    type: extends
  - target: "[[hooks]]"
    type: alternative
sources: []
---

## 一句话定义

外部扩展协议（MCP）+ 可复用的能力包（Skills），让 agent 能力可插拔。

## 核心问题

- MCP 和直接注册工具有什么区别？
- Skill 的粒度怎么定（一个 skill 包含多少能力）？
- 怎么发现和安装第三方扩展？
- 安全边界在哪？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 3: Write channel-remote.md**

```markdown
---
title: Channel & Remote
aliases: [渠道, remote execution, multi-channel]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[runtime-state]]"
    type: uses
sources: []
---

## 一句话定义

多渠道接入（CLI、Web、Slack、IDE）与远程执行能力。

## 核心问题

- 不同渠道的输入输出格式差异怎么抹平？
- 远程执行和本地执行的安全模型有什么区别？
- 渠道间的会话状态怎么同步？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 4: Commit**

```bash
git add wiki/hooks.md wiki/mcp-skills.md wiki/channel-remote.md
git commit -m "wiki: add L1 stubs — Extension (hooks, mcp-skills, channel-remote)"
```

---

### Task 8: Create 12 L1 concept stubs (batch 4 — Reliability)

**Files:**
- Create: `wiki/session-recovery.md`
- Create: `wiki/evaluation-observability.md`

- [ ] **Step 1: Write session-recovery.md**

```markdown
---
title: Session Recovery
aliases: [会话恢复, checkpoint, fault tolerance]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[runtime-state]]"
    type: uses
  - target: "[[query-loop]]"
    type: extends
sources: []
---

## 一句话定义

Agent 断了怎么办 — checkpoint 保存、状态恢复、容错机制。

## 核心问题

- Checkpoint 保存什么（全部状态 vs 增量）？
- 恢复时怎么判断从哪里继续？
- 网络断开、进程崩溃、用户中断分别怎么处理？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 2: Write evaluation-observability.md**

```markdown
---
title: Evaluation & Observability
aliases: [评估, observability, metrics, tracing]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[tool-system]]"
    type: uses
sources: []
---

## 一句话定义

Agent 跑得好不好怎么知道 — 效果评估、运行指标、链路追踪。

## 核心问题

- 怎么定义"agent 做得好"？评估指标是什么？
- 运行时能看到什么（token 用量、工具调用次数、延迟）？
- 怎么做事后分析（trace、replay）？
- 评估是离线跑还是在线跑？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
```

- [ ] **Step 3: Commit**

```bash
git add wiki/session-recovery.md wiki/evaluation-observability.md
git commit -m "wiki: add L1 stubs — Reliability (session-recovery, evaluation-observability)"
```

---

### Task 9: Create lint-rules.yaml

**Files:**
- Create: `schema/lint-rules.yaml`

- [ ] **Step 1: Write basic lint rules**

```yaml
# AI Knowledge Base — Lint Rules
# Used by future linter (Phase B+) to validate wiki quality.

version: "1.0"
updated: "2026-04-06"

rules:
  # Pass 1: Broken links
  broken-links:
    description: 所有 [[wikilink]] 必须指向存在的页面
    severity: error

  # Pass 2: Orphaned pages
  orphaned-pages:
    description: 每个 L2 页面必须被至少一个 L1 页面引用
    severity: warning

  # Pass 3: Missing parent
  missing-parent:
    description: 每个 L2 的 frontmatter.parent 必须指向存在的 L1
    severity: error

  # Pass 4: Frontmatter completeness
  frontmatter-required:
    L1:
      required: [title, category, created, updated, relations, sources]
    L2:
      required: [title, category, parent, source, source_version, confidence, created, updated]
    severity: warning

  # Pass 5: Relation type validation
  relation-types:
    description: relations[].type 必须是 ontology.yaml 中定义的 6 种之一
    valid: [uses, feeds, contradicts, evolved-into, alternative, extends]
    severity: error

  # Pass 6: Naming convention
  naming:
    L1: "^[a-z][a-z0-9-]*\\.md$"
    L2: "^[a-z][a-z0-9-]*--[a-z][a-z0-9-]*\\.md$"
    severity: error

  # Pass 7: Stale content
  stale-content:
    description: updated 超过 180 天的页面标记为需要 review
    threshold_days: 180
    severity: info

  # Pass 8: Duplicate detection
  duplicate-detection:
    description: 新概念入库前向量搜索，>0.85 相似度提示可能重复
    similarity_threshold: 0.85
    severity: warning
```

- [ ] **Step 2: Validate YAML syntax**

```bash
python3 -c "import yaml; yaml.safe_load(open('schema/lint-rules.yaml')); print('YAML valid')"
```

Expected: `YAML valid`

- [ ] **Step 3: Commit**

```bash
git add schema/lint-rules.yaml
git commit -m "schema: add lint rules (8 passes)"
```

---

### Task 10: Full validation and final commit

- [ ] **Step 1: Verify all 12 L1 pages exist**

```bash
expected="prompt-system query-loop tool-system context-management memory-system runtime-state multi-agent hooks mcp-skills session-recovery channel-remote evaluation-observability"
for name in $expected; do
  if [ -f "wiki/${name}.md" ]; then
    echo "OK: wiki/${name}.md"
  else
    echo "MISSING: wiki/${name}.md"
  fi
done
```

Expected: 12 lines all starting with `OK:`

- [ ] **Step 2: Verify all frontmatter is valid YAML**

```bash
for f in wiki/*.md; do
  name=$(basename "$f")
  python3 -c "
import yaml
with open('$f') as fh:
    content = fh.read()
    parts = content.split('---')
    if len(parts) >= 3:
        data = yaml.safe_load(parts[1])
        print(f'OK: $name — {data.get(\"title\", \"NO TITLE\")}')
    else:
        print(f'WARN: $name — no frontmatter')
" 2>&1
done
```

Expected: 13 OK lines (12 L1 pages + _index.md)

- [ ] **Step 3: Verify all wikilinks in _index.md point to existing files**

```bash
python3 -c "
import re, os
with open('wiki/_index.md') as f:
    content = f.read()
links = re.findall(r'\[\[([^\]]+)\]\]', content)
for link in links:
    path = f'wiki/{link}.md'
    status = 'OK' if os.path.exists(path) else 'BROKEN'
    print(f'{status}: [[{link}]] -> {path}')
"
```

Expected: All links show `OK`

- [ ] **Step 4: Verify schema files exist and are valid**

```bash
for f in schema/ontology.yaml schema/lint-rules.yaml; do
  python3 -c "import yaml; yaml.safe_load(open('$f')); print(f'OK: $f')"
done
ls schema/page-templates/L1.md schema/page-templates/L2.md && echo "Templates OK"
```

- [ ] **Step 5: Final summary commit (if any uncommitted changes)**

```bash
git status
# If clean: echo "All committed"
# If changes: git add -A && git commit -m "scaffold: final validation fixes"
```
