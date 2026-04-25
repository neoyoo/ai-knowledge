---
name: kb-ingest
description: "Use when ingesting a new source project into the AI knowledge base. Triggers on phrases like 'ingest X into the knowledge base', 'analyze source project X', 'add X to wiki'. Runs 7-phase autonomous workflow: pre-analysis → parallel L2 writing → L1 semantic patches → coverage scan → tiered review → report generation → self-optimization. All output is isolated in drafts/{source}/ and never modifies wiki/ directly."
---

# kb-ingest — AI 知识库 Ingest Skill

> **职责定位**：写路径。对应 ai-knowledge（读路径）。两个 skill 通过 `wiki/` 目录解耦，互不依赖。

## 输入规范

**必填**：源项目路径（本地目录绝对路径 或 git URL）

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--source-name` | 覆盖自动推断的源名称 | 从目录名或仓库名推断 |
| `--concepts` | 限定分析的概念维度列表（逗号分隔） | 自动推断全部相关维度 |
| `--kb-path` | 知识库根目录绝对路径 | 当前工作目录 |

## 前置依赖验证（Pre-flight Checklist）

**在 Phase 1 开始前必须全部通过，任一失败则输出缺失路径列表并终止：**

```
[ ] schema/ontology.yaml          存在且可读
[ ] schema/page-templates/L1.md   存在且可读
[ ] schema/page-templates/L2.md   存在且可读
[ ] wiki/                         目录存在
[ ] 源路径                         存在且可读（git URL 则可 clone）
```

## 产出目录结构

```
drafts/{source-name}/              # 临时暂存区（ingest 进行中）
├── REVIEW.md                      # review 报告（入口文件）
├── meta.yaml                      # 机器可读元数据
├── wiki/
│   ├── {concept}.md.patch         # L1 语义化增量 patch
│   └── _impl/
│       └── {concept}--{source}.md # L2 完整新页面
└── analysis/
    ├── module-map.md              # 模块 → 概念维度映射
    └── coverage-report.md         # 覆盖率扫描结果
```

## 合并后目录归宿

| 决策 | 操作 |
|------|------|
| **生产级项目** → 合并 wiki | L2 移到 `wiki/_impl/`，应用 L1 patch，`drafts/` 清理 |
| **实验级项目** → 归档 shelf | 整体移到 `shelf/{source}/`，加 `SHELF.md` 说明归档原因，从核心价值点提炼洞察卡写入 `wiki/_insights/` |

**`shelf/` 规范**：不合并但保留完整分析，`SHELF.md` 说明原因和可查阅的价值点。
**`wiki/_insights/` 规范**：轻量洞察卡（1-2 页），提炼单一设计思路，包含：核心洞察 / 设计方案 / 关键代码证据 / 适用场景 / 代价局限 / 来源。

## 错误处理规范

| 错误类型 | 处理方式 |
|---------|---------|
| 前置依赖缺失 | 输出明确错误信息，列出缺失路径，终止执行 |
| 源路径不存在 | 输出错误信息，终止执行 |
| git clone 失败 | 输出错误信息（含 git 错误原因），终止执行 |
| 单个 subagent L2 写入失败 | 记录失败项，继续其他 subagent，在 REVIEW.md 中标注 |
| 覆盖率低于门禁 | 记录 BLOCKED 状态，写入 REVIEW.md，不终止 |
| drafts/ 已存在同名目录 | 输出警告，覆盖写入，不终止 |

---

## Phase 1：预分析

**步骤 1**：运行前置依赖验证。

**步骤 2**：读取 Agent 启动配置（若文件存在）：
- `{kb-path}/schema/ingest-strategies.yaml` → 执行策略指导
- `{kb-path}/schema/ingest-learnings.md` → 参考经验

**步骤 3**：读取 `{kb-path}/schema/ontology.yaml`，解析 14 个概念维度（名称、aliases、定义）。

**步骤 4**：推断 source-name：优先用 `--source-name`；本地路径取末段目录名；git URL 取仓库名。

**步骤 5**：验证源路径；git URL 则 `git clone {url} /tmp/kb-ingest-{timestamp}/`，失败则终止。

**步骤 6**：检查 `{kb-path}/drafts/{source-name}/` 是否已存在；若存在，输出警告并继续。

**步骤 7**：创建产出目录：
```bash
mkdir -p {kb-path}/drafts/{source-name}/wiki/_impl/
mkdir -p {kb-path}/drafts/{source-name}/analysis/
```

**步骤 7.5**：在 `{kb-path}/raw/` 下创建 symlink（若不存在）：
```bash
ln -s {source-path} {kb-path}/raw/{source-name}
```
若 symlink 已存在则跳过（不覆盖）。git URL clone 的临时目录不创建 symlink。

**步骤 8**：扫描源项目顶层目录（排除 `.git/`、`__pycache__/`、`node_modules/`），识别主语言，估算代码规模。

**步骤 9**：用 ontology aliases 做关键词匹配，映射模块 → 概念维度（一对多）。若 `--concepts` 存在，只映射指定维度。

### 产出文件

**`drafts/{source}/analysis/module-map.md`**：写入生成时间、源路径、主语言、代码规模，然后两张表：
- 顶层模块列表：`模块路径 | 主要功能（推断） | 映射概念维度`
- 概念维度覆盖计划：`概念维度 | 相关模块 | 是否派 subagent`
- 排除模块列表（.github/, docs/, tests/, examples/ 等）

**`drafts/{source}/meta.yaml`**（初始状态）：写入 source_name/source_path/ingest_time/kb_path（source_version 留空），其余列表字段置空，status=in-progress，review_summary 三项归零。

---

## Phase 2：深度分析（并行 subagent 派发）

**步骤 1**：从 module-map.md 提取「是否派 subagent = 是」的概念维度，每个维度派一个 subagent，并行度 ≤ 12。

**步骤 2**：运行 `git -C {source-path} log -1 --format="%H %ai"` 获取 source_version；失败则用 `"unknown"`。

**步骤 3**：使用 superpowers:dispatching-parallel-agents 并行派发 subagent（见下方 prompt 模板）。

**步骤 4**：收集成功 L2 文件列表，记录失败项，更新 meta.yaml 的 `concepts_covered` 和 `l2_files`。

### Subagent Prompt 模板

```
你是一个知识库 L2 页面撰写专家。

## 任务
为概念维度「{concept}」分析源项目「{source}」的相关实现，产出完整 L2 页面。

## 输入信息
- 概念维度：{concept}
- 概念定义：{concept-definition}（来自 ontology.yaml）
- 源项目路径：{source-path}
- 相关模块：{module-list}（逗号分隔，如 src/memory/, src/utils/persistence/）
- 源版本：{source-version}
- L2 模板路径：{kb-path}/schema/page-templates/L2.md
- 输出路径：{kb-path}/drafts/{source}/wiki/_impl/{concept}--{source}.md

## 分析要求
1. **深度阅读源码**：读取相关模块的源文件（不是 README 或文档），追踪实际函数调用链
2. **禁止使用伪代码**：关键代码路径章节必须包含真实函数签名和调用链
3. **先读模板**：读取 L2 模板文件，严格按模板结构输出

## 必含章节（四项缺一不可）

| 章节 | 内容要求 |
|------|---------|
| ## 架构分析 | 模块整体设计，主要抽象和数据流向 |
| ## 关键代码路径 | 核心功能的代码级追踪（真实函数签名 + 调用链） |
| ## 设计亮点 | 值得借鉴的创新点或独特选择 |
| ## 局限性 | 已知缺陷、设计权衡代价、使用场景限制 |

## 输出文件 frontmatter 格式

---
title: "{concept}——{source}"
category: L2
source: "{source}"
source_version: "{source-version}"
concept: "{concept}"
created: "{YYYY-MM-DD}"
confidence: high | medium | low
---

写入文件到指定输出路径，然后输出确认：
`[L2 DONE] {concept}--{source}.md 已写入，confidence={high/medium/low}`
```

### L2 质量门禁

所有 L2 文件必须满足：frontmatter 含全部必填字段、包含四个必含章节（有实质内容非占位符）、关键代码路径含真实函数签名、文件命名符合 `{concept}--{source}.md`。不满足则记入 Phase 5 阻断问题。

---

## Phase 3：L1 聚合（语义化 patch）

**不直接修改 wiki/ 下的任何文件**，所有 patch 写入 drafts/ 隔离目录。

**步骤 1**：读取所有 Phase 2 产出的 L2 文件，提取 `concept` 字段确定对应 L1 页面。

**步骤 2**：读取 `{kb-path}/wiki/{concept}.md`；若 L1 不存在，读取 L1 模板作为基础，patch 中注明「新建 L1」。

**步骤 3**：差异分析——对比表中是否已有 {source} 列（无则 OP-1）；设计权衡是否有新场景（无则 OP-2）；是否揭示新概念关系（有则 OP-3）。

**步骤 4**：为每个 L1 页面写入 `{kb-path}/drafts/{source}/wiki/{concept}.md.patch`，包含：
- 文件头：生成时间、基于 L2 路径、目标 L1 路径（已存在 or 新建）
- **OP-1**：在 `## 各家设计对比` 表中新增 `{source}` 列，按实现方式/关键特性/适用场景/主要局限四行填充（来自 L2）
- **OP-2**：在 `## 设计权衡` 场景决策指南中追加场景建议（含原因、代码证据、与已有源对比）；若无新场景，明确写「OP-2: 无新场景决策可补充」
- **OP-3**（可选）：在 `## 关系` 中追加跨概念关系（类型: supports/contradicts/evolved_into/depends_on + 证据）；若无，写「OP-3: 无新跨概念关系发现」

**重要约束**：OP-1 和 OP-2 必须存在，禁止使用「TBD」等占位符。

Phase 3 完成后，更新 meta.yaml 的 `l1_patches` 列表。

---

## Phase 4：覆盖率扫描

### 覆盖率计算规则

```
覆盖率 = 有对应 L2 的顶层目录数 / 有效顶层目录总数

有效目录（计入分母）：src/, core/, lib/, agents/, models/ 等实现目录
排除目录：.github/, docs/, tests/, examples/, scripts/, dist/, build/, config/（纯配置）
覆盖认定：L2 文件中明确分析了该模块（非泛泛提及）；一个 L2 可覆盖多个模块
```

**步骤 1**：逐个判断顶层模块是否被某 L2 文件覆盖，计算初始覆盖率。

**步骤 2**：概念交叉检查——对每个概念维度，用 ontology aliases 及下方补充关键词搜索源码（用 Grep 工具），找出「关键词命中但已有 L2 未提及」的实现，列出遗漏清单。

**附录：概念关键词扩展参考**

| 概念维度 | 补充搜索关键词 |
|---------|-------------|
| memory-system | dream, archive, consolidate, compact, recall, forget, persist |
| query-loop | loop, run, step, turn, cycle, execute, dispatch, agentic |
| tool-system | tool, call, invoke, register, executor, permission, sandbox |
| multi-agent | orchestrat, worker, spawn, delegate, coordinate, handoff, swarm |
| context-management | context, compress, summarize, window, token, truncate, prune |
| hooks | hook, event, lifecycle, pre_tool, post_tool, intercept, middleware |
| mcp-skills | mcp, plugin, extension, server, client, stdio, transport |
| session-recovery | recovery, resume, checkpoint, snapshot, crash, restore |
| channel-remote | channel, websocket, sse, streaming, fastapi, http, interface |
| evaluation-observability | eval, metric, logging, trace, benchmark, observer, span |
| prompt-system | prompt, system_prompt, template, inject, section, priority |
| runtime-state | state, session, storage, persist, serialize, checkpoint |
| finetuning-system | finetune, fine-tune, training, SFT, DPO, GRPO, RL, tuner, reward, trainer, dataset |
| agent-registry-discovery | registry, discovery, register, AgentCard, service-discovery, Nacos, A2A, resolve, address |

**步骤 3**：若发现遗漏，派补漏 subagent（最多一轮），重新计算覆盖率。

**步骤 4**：覆盖率 ≥ 80% → PASS；< 80% → WARN，记录到 coverage-report.md，在 Phase 6 REVIEW.md 中标注 BLOCKED，不终止执行。

### 产出文件：`drafts/{source}/analysis/coverage-report.md`

```markdown
# 覆盖率扫描报告：{source}

**生成时间**: {YYYY-MM-DD HH:MM}

## 汇总

| 指标 | 数值 |
|------|------|
| 有效顶层模块总数 | N |
| 已覆盖模块数 | M |
| 覆盖率 | M/N = XX% |
| 状态 | PASS（≥80%）/ WARN（<80%）|
| 补漏轮次 | 0 / 1 |

## 模块覆盖详情

| 模块路径 | 对应 L2 | 状态 |
|---------|---------|------|

## 排除模块

| 路径 | 排除原因 |
|------|---------|

## 概念交叉检查发现

| 概念维度 | 关键词 | 发现 | 处理 |
|---------|-------|------|------|

## 补漏执行记录

（若无补漏，写：本轮无需补漏）
```

---

## Phase 5：分级 Review 分类

对所有产出质量审查，按严重程度分三级处理。**不暂停执行**，所有问题分类记录到 REVIEW.md。

**审查范围**：所有 L2 文件、所有 L1 patch、coverage-report.md、meta.yaml 完整性。

**级别 A（自动修复）**：可机械判断和修正，直接修复并记入 REVIEW.md「自动修复清单」。
- frontmatter 缺字段 → 补充默认值（如 `confidence: medium`）
- 文件命名不规范 → 重命名为 `{concept}--{source}.md`
- Markdown 标题层级错误 → 修正
- L2 必含章节标题缺失但内容已有 → 补充标准标题
- meta.yaml 字段类型错误（如 `coverage_rate: "85%"`）→ 修正类型

**级别 B（标注待确认）**：需人工判断，写入「待确认问题清单」，标注位置和建议。
- L2 内容较浅（confidence=low 或架构分析 < 300 字）
- OP-3 关系类型存疑
- 场景决策建议缺乏代码证据
- 模块映射到多个概念可能重复
- source_version 为 unknown

**级别 C（阻断）**：可能污染知识库，写入「阻断问题清单」，不暂停执行。
- L2 与已有 L1 描述截然相反
- 设计模式优劣判断与已有结论相反
- L2 缺失四项必含章节中任一项
- 覆盖率 < 80%
- L2 写入失败

Phase 5 完成后，统计 `auto_fixed`、`pending_review`、`blocked` 数量，更新 meta.yaml 的 `review_summary`。

---

## Phase 6：产出报告

### 状态判定逻辑

| 状态 | 判定条件 |
|------|---------|
| `READY_TO_MERGE` | `blocked == 0` 且 `pending_review == 0` 且 `coverage_rate >= 0.80` |
| `NEEDS_REVIEW` | `blocked == 0` 且（`pending_review > 0` 或任何 confidence=low 的 L2）|
| `BLOCKED` | `blocked > 0` 或 `coverage_rate < 0.80` |

> 优先级：BLOCKED > NEEDS_REVIEW > READY_TO_MERGE

### 产出文件一：`drafts/{source}/REVIEW.md`

头部：Ingest 时间/源路径/源版本/kb_path/状态（READY_TO_MERGE | NEEDS_REVIEW | BLOCKED）

章节结构：
- **概览**：指标表（L2 数/L1 patch 数/覆盖率/自动修复/待确认/阻断）+ 产出文件清单（L2 列表含 confidence、L1 patch 列表、分析文件）
- **自动修复清单**：`[x] {文件}：{问题} → {修复}`（若无则说明）
- **待确认问题清单**：每条格式 `[MEDIUM-N] 标题 / 位置 / 问题 / 建议`（若无则说明）
- **阻断问题清单**：每条格式 `[BLOCK-N] 标题 / 位置 / 问题 / 风险 / 建议修复`（若无则说明）

### 产出文件二：`drafts/{source}/meta.yaml`（最终状态）

```yaml
source_name: "{source-name}"
source_path: "{original-path}"
source_version: "{git-hash-or-version}"
ingest_time: "{YYYY-MM-DDThh:mm:ssZ}"
kb_path: "{knowledge-base-root}"
concepts_covered: [concept-1, concept-2]
coverage_rate: 0.85           # 小数，非 "85%" 字符串
l2_files: [wiki/_impl/concept-1--{source}.md]
l1_patches: [wiki/concept-1.md.patch]
status: READY_TO_MERGE        # READY_TO_MERGE | NEEDS_REVIEW | BLOCKED
review_summary:
  auto_fixed: 0
  pending_review: 0
  blocked: 0
notes: ""
```

---

## Phase 7：自优化反馈

核心原则：经验追加自动化，策略更新走人工把关流程。

### 7.1 每次 ingest 完成后（自动执行）

追加经验条目到 `{kb-path}/schema/ingest-learnings.md`（追加到末尾，不修改已有内容）：

```markdown
### {source} — {YYYY-MM-DD}
- 覆盖率: {XX}%，补漏 {N} 轮（{0|1}）
- 发现：{具体观察}
- 教训：{可泛化经验}
- L2 质量分布：high={N} medium={N} low={N}
- 阻断问题：{数量及简述}
- 待确认问题：{数量及简述}
```

### 7.2 策略蒸馏（每 5 次 ingest 后，或手动触发）

1. 读取 `schema/ingest-learnings.md` 所有条目 + `schema/ingest-strategies.yaml` 现有策略
2. 提炼跨多次 ingest 反复出现的可执行规则
3. 写入 `{kb-path}/drafts/strategy-update/ingest-strategies.yaml.patch`：

```yaml
add:
  - id: {策略-id}
    trigger: "{触发条件}"
    rule: "{可执行规则描述}"
    evidence: "{来自哪些 ingest 的证据}"
    added: "{YYYY-MM-DD}"

deprecate: []
```

4. 在 REVIEW.md 结尾追加：`[Phase 7] 策略蒸馏草稿已写入 drafts/strategy-update/`

### 7.3 Agent 启动加载顺序（Phase 1 开始前）

1. 读取 `schema/ingest-strategies.yaml`（若存在）→ 打印：`[kb-ingest] 加载执行策略 N 条`
2. 读取 `schema/ingest-learnings.md`（若存在）→ 打印：`[kb-ingest] 已读取 N 条经验记录`
3. 两个文件均不存在时：输出 `[kb-ingest] 首次运行，策略文件将在本次完成后创建`，继续执行
