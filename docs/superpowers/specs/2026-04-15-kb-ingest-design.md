# kb-ingest — AI 知识库 Ingest Skill 设计 Spec

**日期**: 2026-04-15
**状态**: 已确认，待实现

---

## 1. 项目定位

- **名称**: kb-ingest
- **形态**: 单体 Skill 文件（~300-400 行，模板与检查规则内联）
- **用途**: 全自动 ingest 新源项目到 AI 知识库，产出隔离在 `drafts/` 的完整初稿和 review 报告
- **目标用户**: 任何安装了此 skill 的 Claude Code 用户（通用工具，不绑定特定知识库实例）
- **与 ai-knowledge skill 的关系**: 职责互补，相互独立
  - `ai-knowledge` = 读 / 查询（面向开发决策）
  - `kb-ingest` = 写 / 维护（面向知识库扩充）
- **远程 agent 兼容**: 全程不依赖用户交互输入，所有产出均为文件，适合 remote agent + channel 模式运行

---

## 2. 设计原则

- **最小化输入**: 只需源项目路径（本地目录或 git URL），agent 自主判断覆盖哪些概念维度
- **输出隔离**: 所有产出写入 `drafts/{source}/`，不直接修改 `wiki/`，防止幻觉污染已有内容
- **语义化 patch 而非全量覆盖**: L1 更新以语义化增量 patch 格式表达，人工确认后再 apply
- **Deterministic 执行**: L2 是完整新文件（确定性产出），L1 是增量 patch（人工把关），两层职责清晰
- **覆盖率门禁**: ≥ 80% 源码模块被 L2 覆盖，低于门禁不视为完成
- **不无限循环**: Phase 4 补漏最多执行一轮，防止 agent 陷入死循环
- **前置依赖验证**: 运行前检查知识库结构是否符合预期（ontology、模板、wiki/ 目录存在）

---

## 3. 前置依赖

Skill 运行时，知识库项目中必须存在以下结构：

| 路径 | 说明 |
|------|------|
| `schema/ontology.yaml` | 12 个概念维度定义（含 aliases、definitions） |
| `schema/page-templates/L1.md` | L1 聚合页模板 |
| `schema/page-templates/L2.md` | L2 实现详情页模板 |
| `wiki/` | 现有 L1 页面目录 |

若上述任一路径不存在，skill 应输出明确错误信息并终止，不继续执行。

---

## 4. 输入规范

### 最小输入

```
源项目路径（本地目录绝对路径 或 git URL）
```

### 可选输入

| 参数 | 说明 | 默认值 |
|------|------|--------|
| `--source-name` | 覆盖自动推断的源名称 | 从目录名或仓库名推断 |
| `--concepts` | 限定分析的概念维度列表 | 自动推断全部相关维度 |
| `--kb-path` | 知识库根目录路径 | 当前工作目录 |

---

## 5. 7-Phase 工作流

### Phase 1: 预分析

**目标**: 建立分析计划，验证输入合法性

**步骤**:

1. **验证前置依赖**
   - 检查 `schema/ontology.yaml`、`schema/page-templates/L1.md`、`schema/page-templates/L2.md`、`wiki/` 是否存在
   - 任一缺失 → 输出错误信息，终止执行

2. **读取 ontology**
   - 解析 `schema/ontology.yaml` → 获取 12 个概念维度列表（名称、aliases、定义）

3. **验证源路径**
   - 本地路径：检查目录是否存在且可读
   - git URL：clone 到临时目录（`/tmp/kb-ingest-{timestamp}/`）

4. **防重复检查**
   - 检查 `drafts/{source-name}/` 是否已存在
   - 若存在 → 输出警告（"该源已有 draft，覆盖将清空已有产出"），但不阻断（继续执行，覆盖写入）

5. **扫描源项目目录结构**
   - 列出所有顶层目录/模块
   - 识别主语言（Python/TypeScript/Go 等）
   - 估算代码规模

6. **自动映射概念维度**
   - 对每个模块，用 ontology aliases 做关键词匹配
   - 产出「模块 → 概念维度」映射表
   - 输出 `drafts/{source}/analysis/module-map.md`

**产出**:
- `drafts/{source}/analysis/module-map.md`（模块 → 概念映射）
- `drafts/{source}/meta.yaml`（初始元数据，状态 `in-progress`）

---

### Phase 2: 深度分析（并行 subagent）

**目标**: 为每个相关概念维度产出 L2 页面

**机制**:

- 根据 Phase 1 映射结果，为每个命中的概念维度派一个 subagent
- 每个 subagent 独立运行，互不依赖
- 建议并行度：≤ 12（与概念维度总数一致）

**每个 subagent 的任务**:

1. 读取对应概念维度的源码模块（深度分析，非 README 级别）
2. 读取 `schema/page-templates/L2.md` 模板
3. 产出 L2 页面到 `drafts/{source}/wiki/_impl/{concept}--{source}.md`

**L2 质量门禁（四项必须全部包含）**:

| 必含章节 | 内容要求 |
|---------|---------|
| 架构分析 | 模块整体设计，主要抽象和数据流向 |
| 关键代码路径 | 核心功能的代码级追踪（函数签名 + 调用链，非伪代码） |
| 设计亮点 | 该实现中值得借鉴的创新点或独特选择 |
| 局限性 | 已知缺陷、设计权衡的代价、使用场景限制 |

**L2 页面 frontmatter 规范**:

```yaml
---
title: "{概念维度}——{源项目名}"
category: L2
source: "{源项目名}"
source_version: "{git commit hash 或版本号}"
concept: "{概念维度}"
created: "{YYYY-MM-DD}"
confidence: high | medium | low
---
```

**产出**:
- `drafts/{source}/wiki/_impl/{concept}--{source}.md`（每个概念维度一个 L2 文件）

---

### Phase 3: L1 聚合

**目标**: 产出对现有 L1 页面的语义化增量 patch

**步骤**:

1. 读取所有新产出的 L2 页面
2. 读取 `wiki/` 下对应的现有 L1 页面（若不存在则读取 `schema/page-templates/L1.md` 作为基础）
3. 分析新源的设计与已有内容的差异点
4. 为每个 L1 页面产出语义化 patch

**语义化 patch 格式**（不是 git diff，是自然语言描述的结构化增量）:

```markdown
# L1 语义化 Patch：{concept} ← {source}

## 操作列表

### OP-1: 在"各家设计对比"表中新增一列
位置：## 各家设计对比 → 对比表
操作：新增列 "{source}"
内容：
| 维度 | {source} |
|------|----------|
| 实现方式 | ... |
| 关键特性 | ... |
| 适用场景 | ... |

### OP-2: 在"设计权衡"中补充场景决策
位置：## 设计权衡 → 场景决策指南
操作：追加以下内容
内容：
**当 {具体场景} 时，优先考虑 {source} 的 {做法}**，原因：...

### OP-3: 在"关系"中新增关联
位置：## 关系
操作：追加
内容：
- {source} 的实现 supports [[{related-concept}]]（证据：...）
```

**L1 patch 必须包含**:
- 新对比列（对比表格中补充该源的数据）
- 场景决策补充（基于新源的不同设计，补充适用场景建议）

**产出**:
- `drafts/{source}/wiki/{concept}.md.patch`（每个 L1 页面一个 patch 文件）

---

### Phase 4: 覆盖率扫描

**目标**: 防止注意力遗漏，确保源项目核心模块全部被分析

**步骤**:

1. **目录扫描**
   - 列出源码所有顶层目录/模块（与 Phase 1 保持一致）
   - 逐个确认是否已被某个 L2 文件覆盖
   - 计算覆盖率 = 被覆盖模块数 / 总模块数

2. **概念交叉检查**
   - 对每个 L1 概念维度，用关键词搜索源码（如 memory→dream/archive/consolidate/compact）
   - 找出已有 L2 中未提及的相关实现
   - 列出「遗漏发现」清单

3. **自动补漏**
   - 对遗漏模块，回到 Phase 2 派补漏 subagent 写 L2
   - **最多补一轮**（防无限循环）
   - 补漏完成后重新计算覆盖率

4. **覆盖率门禁**
   - 覆盖率 ≥ 80%：通过，继续 Phase 5
   - 覆盖率 < 80%：记录警告，写入 REVIEW.md 阻断问题，但不停止执行（适配远程 agent）

**产出**:
- `drafts/{source}/analysis/coverage-report.md`（覆盖率扫描结果）

**coverage-report.md 格式**:

```markdown
# 覆盖率扫描报告：{source}

## 汇总
- 总模块数：N
- 已覆盖：M
- 覆盖率：M/N = XX%
- 状态：PASS / WARN（< 80%）

## 模块覆盖详情

| 模块路径 | 对应 L2 | 状态 |
|---------|---------|------|
| src/core/ | agent-loop--{source}.md | 已覆盖 |
| src/memory/ | memory-system--{source}.md | 已覆盖 |
| src/utils/ | —— | 未覆盖（工具类，低优先级）|

## 概念交叉检查发现

| 概念维度 | 关键词 | 发现 | 处理 |
|---------|-------|------|------|
| memory-system | dream/archive | 发现 dream_manager.py 未被 L2 覆盖 | 已派补漏 subagent |

## 补漏执行记录
- 补漏 subagent 1：{concept}--{source}.md（补充 dream_manager 分析）
```

---

### Phase 5: 分级 Review

**目标**: 对所有产出进行质量审查，按严重程度分级处理

**三级分类**:

#### 自动修复（小问题）
条件：可机械判断和修正，无需人工判断
示例：
- frontmatter 缺字段 → 补充默认值
- 文件命名不符合 `{concept}--{source}.md` 规范 → 重命名
- Markdown 标题层级错误 → 修正
- L2 必含章节缺失标题（但内容已有）→ 补充标题

处理方式：直接修复，写入 REVIEW.md「自动修复清单」

#### 标注待确认（中等问题）
条件：需要人工判断，但不影响整体结构
示例：
- 某概念维度覆盖内容较浅（分析不够深入，建议补充）
- 关联关系不确定（是 supports 还是 contradicts？）
- 设计权衡判断依据不足

处理方式：写入 REVIEW.md「待确认问题清单」，标注具体位置和问题描述

#### 阻断（大问题）
条件：可能污染知识库或造成错误决策
示例：
- L2 内容与已有 L1 页面存在明显矛盾（如描述同一组件的实现方式截然相反）
- 重大架构判断分歧（如对某设计模式的优劣判断与已有结论相反）
- L2 页面缺失必含章节（四项质量门禁未全部满足）
- 覆盖率低于 80%

处理方式：写入 REVIEW.md「阻断问题清单」（标红），**不暂停执行**（适配远程 agent）

---

### Phase 6: 产出报告

**目标**: 生成最终 review 报告，完成 meta.yaml 更新

**REVIEW.md 结构**:

```markdown
# Ingest Review 报告：{source}

**Ingest 时间**: {datetime}
**源路径**: {source-path}
**状态**: READY_TO_MERGE | NEEDS_REVIEW | BLOCKED

---

## 概览

| 指标 | 数值 |
|------|------|
| 新增 L2 页面 | N 个 |
| 更新 L1 页面（patch） | M 个 |
| 源码覆盖率 | XX% |
| 自动修复问题 | K 个 |
| 待确认问题 | P 个 |
| 阻断问题 | Q 个 |

---

## 自动修复清单（已处理）

- [ ] {文件路径}：{问题描述} → {修复方式}
- [ ] ...

---

## 待确认问题清单

### [MEDIUM-1] {标题}
- **位置**: {文件路径} § {章节}
- **问题**: {描述}
- **建议**: {处理建议}

---

## 阻断问题清单

> 以下问题建议在 apply 到 wiki/ 前人工确认

### [BLOCK-1] {标题}
- **位置**: {文件路径} § {章节}
- **问题**: {描述}
- **风险**: {若直接合并可能造成的影响}
```

**meta.yaml 最终状态**:

```yaml
source_name: "{source-name}"
source_path: "{original-path}"
source_version: "{git-hash-or-version}"
ingest_time: "{ISO-8601-datetime}"
kb_path: "{knowledge-base-root}"
concepts_covered:
  - agent-loop
  - memory-system
  - ...
coverage_rate: 0.85
l2_files:
  - wiki/_impl/agent-loop--{source}.md
  - ...
l1_patches:
  - wiki/agent-loop.md.patch
  - ...
status: READY_TO_MERGE | NEEDS_REVIEW | BLOCKED
review_summary:
  auto_fixed: K
  pending_review: P
  blocked: Q
```

**最终状态判定**:

| 状态 | 条件 |
|------|------|
| `READY_TO_MERGE` | 无待确认问题，无阻断问题，覆盖率 ≥ 80% |
| `NEEDS_REVIEW` | 有待确认问题，但无阻断问题 |
| `BLOCKED` | 有阻断问题 或 覆盖率 < 80% |

---

## 6. 产出目录结构

```
drafts/{source-name}/
├── REVIEW.md                          # review 报告（入口文件）
├── meta.yaml                          # 机器可读元数据（源信息、覆盖率、状态）
├── wiki/
│   ├── {concept}.md.patch             # L1 语义化增量 patch（每个更新的 L1 一个）
│   └── _impl/
│       ├── {concept}--{source}.md     # L2 完整新页面
│       └── ...
└── analysis/
    ├── module-map.md                  # Phase 1 产出：模块 → 概念维度映射
    └── coverage-report.md             # Phase 4 产出：覆盖率扫描结果
```

**关键设计决策**:

- **L1 用语义化 `.patch`，不用完整文件**: 防止 agent 幻觉污染已有 L1 内容；人工 review patch 比 diff 整文件更高效
- **L2 是完整新文件**: L2 是全新创建，无现有内容风险，直接复制即可
- **`analysis/` 保留过程产物**: 方便 review 时追溯 agent 的判断依据，也是覆盖率审计的凭证
- **`meta.yaml` 机器可读**: 后续可用脚本批量处理、统计、或自动化 apply

---

## 7. 合并流程（Review 通过后）

合并步骤由人工执行（或另一个专门的 apply-draft skill 处理）：

1. **L2 合并**: `drafts/{source}/_impl/*.md` 直接复制到 `wiki/_impl/`
2. **L1 patch apply**: 逐个阅读 `*.md.patch`，根据语义化操作描述手动编辑对应 `wiki/{concept}.md`
3. **验证**: 检查 wiki/ 内部链接是否完整，新 L2 是否被 L1 正确引用
4. **清理**: 确认无误后删除 `drafts/{source-name}/`

---

## 8. 错误处理规范

| 错误类型 | 处理方式 |
|---------|---------|
| 前置依赖缺失 | 输出明确错误信息，列出缺失路径，终止执行 |
| 源路径不存在 | 输出错误信息，终止执行 |
| git clone 失败 | 输出错误信息（含 git 错误原因），终止执行 |
| 单个 subagent L2 写入失败 | 记录失败项，继续其他 subagent，在 REVIEW.md 中标注 |
| 覆盖率低于门禁 | 记录 BLOCKED 状态，写入 REVIEW.md，但不终止（继续产出报告） |
| drafts/ 已存在同名目录 | 输出警告，覆盖写入（不终止） |

---

## 9. Skill 内联规范（实现细节）

### 文件规模目标

- 单体 Skill 文件：约 300-400 行
- 模板内容（L2 frontmatter 格式、patch 格式、REVIEW.md 结构）内联在 skill 中
- 不引用外部模板文件（skill 应自包含）

### Subagent 派发规范

**Phase 2 subagent prompt 模板**（内联在 skill 中）:

```
你是一个知识库 L2 页面撰写专家。

任务：为概念维度「{concept}」分析源项目「{source}」的相关实现，产出 L2 页面。

源码路径：{source-path}
相关模块：{module-list}
L2 模板路径：schema/page-templates/L2.md

要求：
1. 深度阅读上述模块的源码（不是 README 或文档）
2. 必须包含以下四个章节：架构分析、关键代码路径、设计亮点、局限性
3. 关键代码路径必须包含真实函数签名和调用链
4. 输出路径：drafts/{source}/wiki/_impl/{concept}--{source}.md

frontmatter 格式：
---
title: "{concept}——{source}"
category: L2
source: "{source}"
source_version: "{version}"
concept: "{concept}"
created: "{date}"
confidence: high | medium | low
---
```

### 覆盖率计算规则

```
覆盖率 = 有对应 L2 的顶层目录数 / 所有顶层目录总数

排除规则（不计入分母）：
- 纯配置目录（如 .github/、docs/、tests/、examples/）
- 纯工具目录（如 scripts/、tools/、ci/）

判断标准：
- 一个 L2 文件可覆盖多个模块（如果分析内容涵盖了这些模块）
- 覆盖认定：L2 文件中明确提及并分析了该模块
```

---

## 10. 与 ai-knowledge skill 的协作关系

```
kb-ingest skill                    ai-knowledge skill
    │                                      │
    │  产出 drafts/                        │  读取 wiki/
    │  ↓ 人工 review + apply               │
    └──────────────→ wiki/ ←──────────────┘
```

- `kb-ingest` 负责知识入库（写路径）
- `ai-knowledge` 负责知识检索与决策支持（读路径）
- 两个 skill 通过 `wiki/` 目录解耦，互不依赖

---

## 11. Phase 7: 自优化反馈

**目标**: Agent 从每次 ingest 中积累经验，逐步优化分析策略

**核心原则**: Builder 和 Reviewer 职责分离
- **经验文件**（`schema/ingest-learnings.md`）：agent 自由追加，不影响执行逻辑，仅作参考
- **策略文件**（`schema/ingest-strategies.yaml`）：人工把关，走 draft → review 流程

### 11.1 每次 ingest 完成后（自动）

回顾本次表现，追加经验到 `schema/ingest-learnings.md`：

```markdown
### {source} — {date}
- 覆盖率: XX%，补漏 N 轮
- 发现：{具体观察，如"该项目把 memory 实现藏在 utils/persistence/ 下"}
- 教训：{可泛化的经验，如"Python 项目的 utils/ 目录不能跳过"}
- 阻断问题：{数量及简述}
- L2 质量分布：{high/medium/low 各几个}
```

### 11.2 每 5 次 ingest 后（或手动触发）

1. 读取所有经验条目
2. 提炼可执行策略规则
3. **策略走 draft → review 流程**：写入 `drafts/strategy-update/ingest-strategies.yaml.patch`
4. 人工确认后 apply 到 `schema/ingest-strategies.yaml`
5. 清理已被策略覆盖的经验条目

**策略文件格式**（`schema/ingest-strategies.yaml`）：

```yaml
version: "1.0"
strategies:
  - id: python-utils-scan
    trigger: "源项目主语言 == Python"
    rule: "utils/ 和 lib/ 目录必须深度扫描，不列入覆盖率排除名单"
    source: "mempalace ingest, hermes ingest"
    added: "2026-04-20"
    
  - id: event-driven-loop
    trigger: "检测到 EventEmitter/pubsub 模式"
    rule: "query-loop L2 分析需特别标注事件驱动 vs 传统循环的差异"
    source: "project-x ingest"
    added: "2026-05-01"
```

### 11.3 Agent 启动时加载顺序

1. 读取 `schema/ingest-strategies.yaml`（如存在）→ 作为执行策略指导 Phase 1-6
2. 读取 `schema/ingest-learnings.md`（如存在）→ 作为参考经验
3. 进入 Phase 1

---

## 12. 实现优先级

| 优先级 | 内容 | 说明 |
|--------|------|------|
| P0 | Phase 1 预分析 + 前置验证 | 阻断后续所有步骤，必须稳健 |
| P0 | Phase 2 L2 并行写入 | 核心价值产出 |
| P0 | Phase 6 REVIEW.md 产出 | 用户入口，必须存在 |
| P1 | Phase 3 L1 语义化 patch | 重要但可先产出粗版 |
| P1 | Phase 4 覆盖率扫描 | 质量门禁，建议同步实现 |
| P2 | Phase 5 分级 review 逻辑 | 可先实现自动修复部分 |
| P2 | meta.yaml 最终状态计算 | 机器可读，便于后续自动化 |
| P2 | Phase 7 自优化反馈 | 经验追加为 P2，策略提炼为 P3（需积累足够数据） |

---

## 附录 A：概念维度参考（来自 schema/ontology.yaml）

以下 12 个概念维度为标准分析单元（以 ontology.yaml 为准，此处仅为参考）：

1. Prompt System（prompt-system）
2. Query Loop（query-loop）
3. Tool System（tool-system）
4. Context Management（context-management）
5. Memory System（memory-system）
6. Runtime State（runtime-state）
7. Multi-Agent（multi-agent）
8. Hooks（hooks）
9. MCP & Skills（mcp-skills）
10. Session Recovery（session-recovery）
11. Channel & Remote（channel-remote）
12. Evaluation & Observability（evaluation-observability）

---

## 附录 B：关键词映射示例（覆盖率扫描用）

| 概念维度 | 搜索关键词 |
|---------|----------|
| memory-system | memory, dream, archive, consolidate, compact, recall, forget |
| agent-loop | loop, run, step, turn, cycle, execute, dispatch |
| tool-system | tool, call, invoke, register, execute, permission |
| multi-agent | orchestrat, worker, spawn, delegate, coordinate, handoff |
| context-memory | context, compress, summarize, window, token, truncate |
| extension-model | mcp, plugin, extension, server, client, stdio |

---

*文档状态：已确认设计，待实现*
