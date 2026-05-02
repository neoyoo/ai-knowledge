# AI Engineering Knowledge Base

## 项目定位

全域 AI 工程知识库 -- 受 Karpathy LLM Wiki 启发，覆盖整套 AI 工程领域的结构化知识系统。

**核心理念**: 知识是一张网不是抽屉。跨域连接才是价值所在。

## 知识覆盖范围

| 领域 | 包含内容 |
|-----|--------|
| Agent Orchestration | Loop、状态机、决策策略 |
| Prompt Engineering | 技巧、框架、评估方法 |
| Model Training & Tuning | 微调、量化、DPO |
| Tool Ecosystem | 工具调度、生态设计、MCP、扩展 |
| Infrastructure | 推理优化、框架、部署 |
| Evaluation | 基准、评估框架、指标 |
| Context & Memory | 长对话、压缩、检索增强 |

## 目录结构

| 目录 | 作用 | 内容性质 | 合入规则 |
|-----|------|---------|---------|
| `raw/` | 原始材料区 | 每个源项目的 git clone / symlink（agentscope、claude-code、deer-flow、hermes、mempalace、neoagent、openharness、simplemem …） | 不可变。只读取，不修改 |
| `wiki/` | 正式知识主干 | 顶层 `*.md` 为 L1 概念页（按架构维度组织，如 `tool-system.md`、`sandbox-isolation.md`）；`_impl/<concept>--<source>.md` 为 L2 per-source 实现分析；`_insights/<source>--<design>.md` 为单源精彩设计抽出；`_patterns/<slug>.md` 为**跨 L1 概念的组合模式**（可迁移架构语汇）；`_index.md` 总索引 | 只放通过质量门禁的成熟内容 |
| `ideas/` | **好想法 inbox**（新） | 任何"值得但未落地"的想法，按功能域放入 `ideas/<domain>.md`；一个域文件可包含多条想法，每条想法 section 开头标 `**status**: inbox\|incubating\|promoted\|dead`，并尽量标 `**potential_target**:`。定期 review 决定升级到 wiki/_patterns/、wiki/_insights/、cookbook/、projects/ 或宣布死亡 | 轻格式、低门槛、定期整理 |
| `cookbook/` | 实操层（"怎么动手做"） | `prompts/`（patterns、templates、anti-patterns）+ `tools/`（definitions 按功能分类、patterns、anti-patterns）+ `skills/`（patterns、templates、anti-patterns）| 实操模板和可复制 schema，和 wiki 的"为什么这样设计"互补 |
| `schema/` | 配置与规范 | `ontology.yaml`（关系类型定义）、`lint-rules.yaml`（linter 规则）、`ingest-strategies.yaml`（ingest 策略）、`page-templates/`（L1.md / L2.md 模板）、`ingest-learnings.md`（每次 ingest 积累的经验） | 仅修改规范、不放知识内容 |
| `shelf/` | 归档区（不合入主干） | 质量不够、架构不稳定、尚未成熟的源。每个项目带 `SHELF.md` 说明归档原因。完整 L2 分析保留以便查阅 | 不合并到 `wiki/`。L2 页保留可查，L1 patch 仅作参考不应用 |
| `drafts/` | kb-ingest 工作中区 | 在跑/刚完成但未决定去向的 ingest 产物 | 完成后按质量决策：合入 `wiki/` 或搬去 `shelf/`，搬完应清空 |
| `docs/` | 知识库元文档 | `sdk-kb-alignment.md`（SDK 与 KB 的版本对齐追踪）等。**不是** 被 ingest 的知识，**是** 关于 KB 项目本身的说明 | 维护 KB 运营所需的表格、对齐记录 |
| `.obsidian/` | Obsidian vault 配置 | vault 设置、图谱视图、插件配置 | Obsidian 自管，不手工维护 |
| `.claude/` | Claude Code 项目配置 | 本项目专用 claude skills、settings | 按 Claude Code 约定维护 |

### 目录关系示意

```
raw/       ──  ingest  ──→  drafts/  ──  review  ──┬─→  wiki/     (主干，被权威引用)
(不可变源)                   (工作中)               ├─→  wiki/_insights/ (单源精彩设计)
                                                    └─→  shelf/    (保留但不入主干)
                                                         │
                                                         └─→ SHELF.md（候选提升项 → ideas/）

ideas/      ←── 闪念入口 ──→  wiki/_patterns/  (跨概念组合模式)
(好想法 inbox)                 wiki/_insights/  (单源洞察)
(定期 review 决定升级)         practice/        (我们做过的，待建)
                                cookbook/        (通用模板)

schema/     ←──── 规则约束 ────→   wiki/ / cookbook/
(ontology、lint、templates、ingest-learnings)

cookbook/   ←── 实操互补 ──→   wiki/
("怎么做")                      ("为什么这样")

docs/       关于 KB 本身的记录（sdk-kb-alignment 等），不属于知识内容
```

## 架构

### 三层结构
1. **Raw Sources** -- 不可变的原始材料（源码、文章、论文）
2. **Wiki** -- 结构化 Obsidian markdown 页面库（按概念维度组织）
3. **Schema** -- 配置控制层（知识分类、引用规则、页面模板）

### 概念维度（页面组织方式）
按架构概念而非来源组织：Agent Loop / Tool System / Context & Memory / Multi-Agent / Prompt Architecture / Extension Model / Reliability / Channel & Interface

### 三个核心操作
- **ingest** -- 原始源 → 提取架构洞察 → 映射概念维度 → 更新概念页 → 标注关系
- **query** -- 综合查询，汇聚各家设计对比，给出决策建议
- **lint** -- 周期性质量维护（断链、孤立、矛盾、过期、去重）

### 核心设计原则
- **Deterministic Executor** -- LLM 只输出结构化意图（KEEP/UPDATE/MERGE/SUPERSEDE/ARCHIVE），由 executor 确定性执行，杜绝幻觉污染
- **8-pass Linter** -- 断链/孤立/矛盾/过期/向量补边/TODO/去重/元数据校验
- **类型化关系** -- supports/contradicts/evolved_into/depends_on 等语义关系
- **增量去重** -- 新概念入库前向量搜索，>0.85 相似度自动去重或合并

## 落地形态

- **前端**: Obsidian vault（markdown、wikilinks、反向链接、Graph View）
- **后端**: Ingest 管道（源码/文章 → 提取+结构化+关联 → Obsidian markdown）
- **版本控制**: git

## 上下游关系

```
AI 工程知识库 (概念维度、架构对比、设计权衡)
  ↓ (upgrade 提案)
create-agent-skill (skill 模块、工程实现)
  ↓ (使用)
用户的 agent 项目
```

## 工作准则

- **质量优先**: 可以慢不能乱，生产级标准
- **深度源码分析**: 知识来源必须是源码级深度分析，不是看 README/博客
- **蒸馏设计**: 提取设计本质，不照搬代码
- **跨域关联**: 每条知识都要考虑与其他领域的连接
- **溯源追踪**: 每条知识标注来源、版本、置信度
- **价值优先于数量**: 知识库服务于智能体开发决策，不是为了图好看而堆页面

## Ingest SOP（每次新增源项目必须遵守）

### 前置检查（扩源前）
1. 现有 L1 的"设计权衡"是否已写深？（必须有方案对比 + 场景决策 + 常见陷阱）
2. 新源项目的设计理念是否与已有源**有本质差异**？同质化复刻不值得加
3. 新源能为 L1 对比表带来什么新视角？说不清楚就不加

### Ingest 流程
1. **深读源码** → 产出 L2 页面（`wiki/_impl/概念--来源.md`）
2. **覆盖率扫描**（防遗漏，见下方）
3. **更新 L1 对比表** → 加新列
4. **深化设计权衡** → 基于新源的不同设计，补充场景决策指南（这步不能跳过！）
5. **关联检查** → 新源是否揭示了概念间的新关系？

### 覆盖率扫描（防注意力遗漏）
subagent 分析源码时会因注意力限制遗漏功能。每次 ingest 完成后必须执行：
1. **目录扫描** → 列出源码所有顶层目录/模块，逐个确认是否已被 L2 覆盖
2. **概念交叉检查** → 对每个 L1 概念，用关键词（如 memory→dream/archive/consolidate/compact）搜索源码，找未被提及的相关实现
3. **遗漏修补** → 发现遗漏后补充到对应 L2 页面，并更新 L1

### 质量门禁
- L2 页面必须有架构分析 + 关键代码路径 + 设计亮点 + 局限性
- L1 设计权衡必须包含：方案对比表 + 场景决策指南 + 常见陷阱
- 禁止只加 L2 不更新 L1（L2 是手段，L1 才是价值）

## 工作分工

| 角色 | 职责 |
|-----|------|
| Neo | 选方向、挑源、问问题、做决策 |
| Claude | 深挖分析、结构化知识、维护一致性、确保跨域关联 |
| Subagent | 执行具体工作（源码分析、文件写入、搜索等） |

## 页面 Schema 规范（待细化）

每个概念页应包含:
- **frontmatter** -- 标题、类别、标签、来源列表、最后更新时间
- **概述** -- 一句话定义 + 在 AI 工程中的位置
- **各家设计对比** -- 不同项目/论文对同一概念的实现方式
- **设计权衡** -- 适用场景、优劣分析
- **关系** -- 与其他概念页的类型化链接
- **来源** -- 所有引用的原始来源及版本
