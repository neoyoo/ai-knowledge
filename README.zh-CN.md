# ai-knowledge

> 结构化 Obsidian 知识库 + 本体论——以概念维度为轴组织 AI 工程知识，而非以来源为轴。

[English](README.md) | [中文]

## 是什么

ai-knowledge 是一个覆盖 AI 工程领域的个人知识库：Agent 架构、Prompt 工程、工具系统、上下文管理、多 Agent 编排、MCP、评估、微调等。以 Obsidian vault 形式组织，配套形式化本体——14 个横跨各来源的概念维度，每个维度有 L1 概念页、L2 各源实现分析、跨概念组合模式页，以及轻量想法 inbox。

知识库公开是为了透明度，不是一个开箱即用的产品，也不是精选阅读列表。其结构反映了一个工程师在快速演进领域中理解、沉淀设计决策、避免重复调研的具体工作流。

配套的 `kb-ingest` skill 自动化新源项目的摄入流程：深度源码分析、L2 页面生成、L1 语义 patch、覆盖率扫描——所有产出先暂存在 `drafts/`，review 通过后再合入 wiki。

## 设计理念

**知识是一张网，不是抽屉。** 页面之间通过类型化关系连接（`supports`、`contradicts`、`evolved_into`、`depends_on`）。孤立的事实廉价，跨概念连接才是价值所在。

**按概念维度组织，而非按来源组织。** 关于上下文压缩的页面把所有相关项目汇集在一处。"neoagent 怎么做" 和 "Claude Code 怎么做" 并排对比，远比两个独立项目文件夹有用。

**摩擦日志驱动知识库自我进化。** 每次 KB 无法回答问题——内容缺失、深度不够、答非所问——都记录到 `projects/<proj>/kb-friction.md`。累积的摩擦条目是 KB 诚实的改进信号。没有这个机制，知识库会在看起来健康的状态下悄悄腐烂。

**确定性执行器，而非大模型自由编辑。** Ingest 管道产出结构化意图操作（`KEEP / UPDATE / MERGE / SUPERSEDE / ARCHIVE`），由确定性 executor 执行。LLM 提议，executor 落地。这防止幻觉漂移随时间污染 wiki。

**想法有生命周期。** 零散观察进入 `ideas/` 标记 `status: inbox`，经孵化后升级到 `wiki/_patterns/` 或 `wiki/_insights/`，或明确标记为死亡并保留死亡原因。没有内容会悄悄消失。

## 快速开始

主要入口是 `ai-knowledge` Claude Code skill。它将查询路由到正确的页面，强制执行"先查 KB"纪律，并自动记录摩擦。

```bash
# 将 skill 链接到 Claude Code
ln -s /path/to/ai-knowledge/skills/ai-knowledge ~/.claude/skills/ai-knowledge
```

启用后，Claude Code 在处理 Agent 架构、设计决策、跨项目对比等任务前会先查询知识库。典型触发短语：

- "上下文压缩策略怎么设计？"
- "对比各项目的 session 恢复方案。"
- "有没有把 free/recall 和工具元数据组合起来的 pattern？"

向 KB 摄入新源项目，使用 `kb-ingest` skill（见 `skills/kb-ingest/SKILL.md`）。

## 目录结构

```
ai-knowledge/
├── wiki/               L1 概念页 + L2 各源实现分析
│   ├── *.md            14 个 L1 概念页（query-loop、tool-system、context-management …）
│   ├── _impl/          L2 页：<concept>--<source>.md
│   ├── _insights/      单源精彩设计抽出
│   └── _patterns/      跨概念组合模式（可迁移架构语汇）
├── cookbook/           实操层（prompts / tools / skills × patterns / templates / anti-patterns）
├── ideas/              想法 inbox — status: inbox | incubating | promoted | dead
├── projects/           各项目决策日志和摩擦日志
├── raw/                源项目 symlink（只读，不修改）
├── schema/             本体论、lint 规则、ingest 策略、页面模板
├── shelf/              归档源——完整分析保留，不合入 wiki
├── drafts/             Ingest 进行中产物（review 前暂存区）
└── docs/               KB 元文档（sdk-kb-alignment 等）
```

### 本体论

Schema 定义了 14 个概念维度：

| 维度 | 覆盖内容 |
|---|---|
| `query-loop` | Agent 主循环、状态机、对话轮次 |
| `prompt-system` | 动态 Prompt 组装、section、优先级 |
| `tool-system` | 工具调度、注册表、权限模型 |
| `context-management` | 上下文窗口、压缩、free/recall |
| `memory-system` | 跨会话持久记忆 |
| `runtime-state` | 会话状态、生命周期、检查点 |
| `session-recovery` | 断点续传、崩溃恢复、容错 |
| `multi-agent` | Orchestrator/Worker 模式、任务委派 |
| `hooks` | 事件 Hook、生命周期拦截 |
| `mcp-skills` | MCP 协议、Skill 扩展点 |
| `channel-remote` | HTTP Channel、SSE 流式、FastAPI |
| `evaluation-observability` | 评估框架、指标、链路追踪 |
| `finetuning-system` | SFT、DPO、GRPO、训练管道 |
| `agent-registry-discovery` | Agent 注册发现、服务发现、AgentCard |

完整定义和类型化关系类型见 `schema/ontology.yaml`。

### 知识流向

```
raw/          →  ingest  →  drafts/   →  review  →  wiki/         (主干)
（不可变）                （暂存区）               wiki/_impl/    （L2 分析）
                                                   wiki/_insights/
                                                   shelf/          （归档，不合入）

ideas/                                 →  wiki/_patterns/
（inbox → incubating → promoted/dead）    wiki/_insights/
                                          cookbook/

projects/<proj>/kb-friction.md   →  指引下一步 ingest 优先级
```

## 与 kb-ingest 配合使用

`kb-ingest` skill 自动化新源项目的摄入。流程分 7 个阶段：预分析、并行 L2 页面撰写（最多 12 个并发 subagent）、L1 语义 patch 生成、覆盖率扫描（目标 ≥ 80%）、分级 Review、报告生成、自优化反馈。所有产出先落在 `drafts/{source}/`，review 通过后才触碰 `wiki/`。

```bash
ln -s /path/to/ai-knowledge/skills/kb-ingest ~/.claude/skills/kb-ingest
```

在 Claude Code 中触发：`"Ingest /path/to/some-project into the knowledge base"`。

## 配套 Skill

本仓库附带两个 skill：

- `skills/ai-knowledge/SKILL.md` — 读路径。查询知识库、定位相关页面、记录摩擦。日常使用的权威指南。
- `skills/kb-ingest/SKILL.md` — 写路径。自主 7 阶段摄入管道。新增源项目的完整协议。

将两个 skill 链接或复制到 `~/.claude/skills/` 即可在 Claude Code 中启用。

## 致谢

- **[Obsidian](https://obsidian.md)** —— vault 基底。markdown + backlink + 本地优先存储让这种规模的知识图谱真正可用。
- **卢曼（Niklas Luhmann）卡片盒方法** —— 按概念维度切（不是按来源/时间切），卡片之间是类型化关系，想法从 inbox 到 archive 有完整生命周期。
- **Andrej Karpathy** —— 这个 vault 试图地图化的整个领域的核心框架：LLM 作为新计算界面、"Software 3.0"、以及"先重新推导再去 Google"的纪律。很多 wiki 页是从 Karpathy 提的问题出发的。
- **[Claude Code](https://claude.com/claude-code)** —— skills 系统作为"按需加载知识"的接口，被我们镜像到 `kb-ingest` 7 阶段流水线（drafts → review → wiki）里。

每一页 wiki 都追溯到具体源项目。每一个跨概念模式都来自把那些源放在一起读。

## 许可证

MIT。见 `LICENSE`。

## 状态

个人知识库，公开仅为透明度。不作为开箱即用产品——本体论和模式反映的是作者在 AI Agent 开发中的具体工作流。内容成熟度因源而异：`wiki/` 页面已经过质量门禁；`shelf/` 条目为归档内容，局限性已在各自的 `SHELF.md` 中注明。
