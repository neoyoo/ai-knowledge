# AI Engineering Knowledge Base — Ontology Design Spec

> 日期：2026-04-06
> 状态：已确认，待实施
> 参与者：Neo + Claude

## 1. 项目范围

知识库聚焦于**智能体开发**领域，以"如何构建智能体"为核心视角。覆盖 agent loop、tool system、context management、harness（运行时外壳）、multi-agent、prompt architecture 等。

不是泛 AI 工程全覆盖。

## 2. 三层架构

| 层级 | 目录 | 内容 | 可变性 |
|------|------|------|--------|
| Raw Sources | `raw/` | 原始材料（源码、文章、论文） | 不可变 |
| Wiki | `wiki/` | 结构化知识页面（L1 + L2） | 持续更新 |
| Schema | `schema/` | 配置控制（ontology、模板、lint 规则） | 低频变更 |

## 3. L1/L2 聚合层设计

### 核心思路

借鉴渐进式披露（Progressive Disclosure）：

- **L1 聚合概念页**（~1-2K tokens）：跨项目的概念摘要、各家对比表、设计权衡。Graph View 中的节点。
- **L2 具体实现页**（~2-5K tokens）：单个项目的源码级深度分析。只链回父 L1，不互相链接。

### 关键规则

- L2 之间不互连 — 跨组件关系在 L1 层表达
- Ingest 流向：raw source → 产出 L2 → 合成/更新 L1
- 查询路径：L1 → L2 → raw sources（三级 drill-down）
- 去重发生在 L1 层面

## 4. 文件结构

```
ai-knowledge/
├── CLAUDE.md
├── raw/                           ← 原始材料（不可变）
│   ├── claude-code/
│   ├── openai-agents-sdk/
│   └── ...
├── wiki/                          ← 知识库主体
│   ├── _index.md                  ← 全局入口，概念地图
│   ├── prompt-system.md           ← L1 聚合概念页
│   ├── query-loop.md
│   ├── tool-system.md
│   ├── context-management.md
│   ├── memory-system.md
│   ├── runtime-state.md
│   ├── multi-agent.md
│   ├── hooks.md
│   ├── mcp-skills.md
│   ├── session-recovery.md
│   ├── channel-remote.md
│   ├── evaluation-observability.md
│   └── _impl/                     ← L2 具体实现页
│       ├── tool-system--claude-code.md
│       ├── tool-system--openai-agents-sdk.md
│       └── ...
└── schema/                        ← 配置控制层
    ├── ontology.yaml              ← 概念定义 + 关系类型
    ├── page-templates/            ← L1/L2 页面模板
    └── lint-rules.yaml            ← Linter 规则
```

### 命名规则

- L1：`kebab-case.md`（如 `tool-system.md`）
- L2：`概念--来源.md`，双横线分隔（如 `tool-system--claude-code.md`）
- Raw：按项目名建文件夹

## 5. 12 个初始 L1 概念

| # | 文件名 | 概念 | 一句话定义 |
|---|--------|------|----------|
| 1 | prompt-system.md | Prompt System | 动态组装发给模型的 prompt |
| 2 | query-loop.md | Query Loop | Agent 主循环 — 发请求、拿结果、判断下一步 |
| 3 | tool-system.md | Tool System | 发现、选择、调用、管理外部工具 |
| 4 | context-management.md | Context Management | 管理有限的 context window |
| 5 | memory-system.md | Memory System | 跨会话的持久记忆 |
| 6 | runtime-state.md | Runtime State | 运行时状态容器与生命周期 |
| 7 | multi-agent.md | Multi-Agent | 多 agent 协作 — 拆任务、分发、汇总 |
| 8 | hooks.md | Hooks | 事件驱动的扩展点 |
| 9 | mcp-skills.md | MCP & Skills | 外部扩展协议 + 可复用能力包 |
| 10 | session-recovery.md | Session Recovery | 断点恢复与容错 |
| 11 | channel-remote.md | Channel & Remote | 多渠道接入与远程执行 |
| 12 | evaluation-observability.md | Evaluation & Observability | 效果评估 + 运行可观测 |

### 扩展规则

新增 L1 维度的常规条件：
1. 至少 3 个不同项目/来源对该概念有独立实现
2. 不能被已有 L1 概念覆盖（否则归为 L2）
3. 需要 Neo 和 Claude 讨论审核确认

**例外**：设计足够优秀/创新的，可免除条件 1，单源也可直接提为 L1。

## 6. 关系类型

L1 概念之间是**平级网络关系**，没有上下级。层级只存在于 L1→L2 的聚合关系中。

| 关系 | 含义 | 例子 |
|------|------|------|
| A 用了 B | 运行时调用 | Query Loop → Tool System |
| A 给 B 提供数据 | 数据流向 | Memory → Context Management |
| A 和 B 思路相反 | 理念冲突 | 长 context vs RAG |
| A 变成了 B | 技术演化 | Function Calling → Tool Use |
| A 和 B 是替代方案 | 同一问题不同解法 | ReAct vs Plan-and-Execute |
| A 给 B 加了新能力 | 扩展 | MCP 扩展 Tool System |

## 7. L1 页面模板

```markdown
---
title: {{概念名}}
aliases: [中文别名, 英文别名]
category: L1
created: YYYY-MM-DD
updated: YYYY-MM-DD
relations:
  - target: "[[相关概念]]"
    type: 用了 | 提供数据 | 思路相反 | 变成了 | 替代方案 | 加了新能力
sources:
  - 项目名1
  - 项目名2
---

## 一句话定义

这个组件是什么，在 agent 架构中扮演什么角色。

## 核心问题

这个组件要解决什么？
- 问题 1
- 问题 2

## 各家对比

| 维度 | 项目 A | 项目 B | ... |
|------|--------|--------|-----|
| 维度 1 | ... | ... | ... |

## 设计权衡

- **选择 A vs 选择 B**：优劣分析...

## L2 详情

- [[概念--项目A]]
- [[概念--项目B]]
```

## 8. L2 页面模板

```markdown
---
title: "{{概念}} — {{来源}}"
category: L2
parent: "[[概念名]]"
source: 项目名
source_version: "版本号"
confidence: high | medium | low
created: YYYY-MM-DD
updated: YYYY-MM-DD
---

## 概述

一段话概括该项目对这个概念的实现方式。

## 架构分析

### 子话题 1
具体实现分析...

### 子话题 2
具体实现分析...

### 关键代码路径
- `路径1` — 说明
- `路径2` — 说明

## 设计亮点

- 亮点 1

## 局限性

- 局限 1

## 来源

- 源码版本：xxx
- 分析深度：源码级 | 文档级 | 博客级
```

## 9. Ingest 工作流

### 当前阶段：手动模式（Phase A）

```
Neo 指定分析目标（如 "分析 OpenAI Agents SDK 的 tool system"）
    ↓
Claude 派 subagent 深读源码
    ↓
Subagent 产出：
  ① L2 页面草稿（wiki/_impl/tool-system--openai-agents-sdk.md）
  ② L1 更新建议（wiki/tool-system.md 对比表加一行）
    ↓
Claude 审查质量，确认后写入
    ↓
检查：关系标注完整？与已有 L2 有重复？
```

### 未来演进

- **Phase B**：流程固化为 ingest skill，标准化分析步骤
- **Phase C**：全自动管道，给项目地址批量分析

## 10. 与 Obsidian 的交互

- 将 `wiki/` 文件夹用 Obsidian 打开（Open folder as vault）
- 所有 `.md` 文件自动成为 Obsidian 页面
- `[[wikilink]]` 自动生成链接和反向链接
- frontmatter YAML 支持搜索和过滤
- Graph View 展示 L1 节点和关系网络

## 11. 来源项目（已分析 / 计划分析）

### 已深度分析
- Claude Code（create-agent-like-claude 项目中已完成 11 模块蒸馏）

### 计划分析
- OpenAI Agents SDK
- Dify
- LangGraph
- （更多待定）

## 12. 与 create-agent-like-claude 的关系

| | create-agent-like-claude | ai-knowledge |
|---|------------------------|--------------|
| 视角 | 单项目蒸馏（Claude Code） | 跨项目对比 |
| 产出 | 可执行的 skill 集 | 结构化知识库 |
| 关系 | 是 ai-knowledge 的第一个 raw source | 是更广泛的知识网络 |

create-agent-like-claude 的 11 个模块分析可直接作为 ai-knowledge 的第一批 L2 内容导入。
