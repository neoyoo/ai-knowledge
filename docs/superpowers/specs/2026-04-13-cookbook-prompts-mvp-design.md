# Cookbook Prompts MVP 设计文档

> 日期: 2026-04-13
> 状态: Draft
> 作者: Neo + Claude

## 1. 背景与动机

ai-knowledge 项目的核心定位是 **Agent 开发百科全书**，主要消费者是 LLM（通过 skill 导航按需读取），人类也能直接阅读。

当前知识库已有 wiki/ 层（架构原理、设计权衡、跨源对比），但缺少实操层——开发者（或 LLM）知道"为什么这样设计"，却找不到"怎么动手做"。

cookbook/ 就是百科全书缺的下半部分。

### 项目架构全景

```
ai-knowledge/
├── wiki/        ← 概念层：为什么这样设计（架构原理、跨源对比、权衡分析）
├── cookbook/     ← 实操层：怎么动手做（pattern、template、recipe）
├── raw/         ← 原始素材（源码、文章、项目）
├── schema/      ← 配置控制
└── docs/        ← 项目文档
```

### Skill 架构

```
skill（轻量导航员）
  → "做 X 决策时，读 wiki/Y"
  → "写 prompt 时，读 cookbook/prompts/Z"
  → "搭 tool 时，读 cookbook/tools/W"

ai-knowledge/（百科全书本体）
  → wiki/ + cookbook/ = 不同章节
  → LLM 按需读取具体页面
```

Skill 不包含知识内容，只教 LLM 怎么查。内容和 skill 完全解耦。

## 2. MVP 目标

用 5 个 prompt pattern 验证：

1. **页面 Schema** — 结构是否对 LLM 检索和消费友好
2. **内容粒度** — 每个 pattern 该写多深、多长
3. **升维效果** — 相比原始素材（Prompt Engineering Guide）增加了多少决策价值
4. **可发现性** — frontmatter + 摘要能否支持快速定位

验证通过后，批量铺开剩余 pattern 并扩展到 tools/、skills/ 等板块。

## 3. 原始素材

### Prompt Engineering Guide

- GitHub 70k+ star，DAIR.AI 维护
- 本地路径：/Users/neo/Desktop/project/git/Prompt-Engineering-Guide
- 134 英文页，覆盖 18 种 technique、30+ prompt 示例、20+ 模型页
- 优点：覆盖广、学术锚定、有实际 prompt + output
- 不足：教科书风格（解释概念为主）、缺场景决策、缺组合方案、Prompt Hub 条目太薄

### 升维策略

不搬运，在原始素材基础上增加：

| 原素材有的 | 我们加的 |
|-----------|---------|
| 技巧是什么 | 什么时候用、什么时候不用 |
| 一个例子 | 多个变体 + 模板（带 placeholder） |
| 学术论文引用 | 工程实践经验 + 模型差异 |
| 独立介绍每种技巧 | 技巧间的组合关系和选择指南 |
| 通用解释 | Agent 开发场景的具体适配 |

## 4. MVP Pattern 选择

| # | Pattern | 为什么选 | 升维空间 |
|---|---------|---------|---------|
| 1 | **Chain-of-Thought (CoT)** | 基础推理能力，几乎所有 agent 都用 | 原 guide 只讲触发方式，缺"何时不用"、模型表现差异、与其他 pattern 组合 |
| 2 | **ReAct** | Agent 核心循环（推理+行动） | 原 guide 偏学术，缺实际框架中的工程变体和适配 |
| 3 | **Structured Output** | Tool calling / 解析必备 | 原 guide 几乎没有专门讲，从实践提炼 |
| 4 | **Prompt Chaining** | 任务分解与编排 | 原 guide 只有 QA 例子，缺 agent pipeline 场景 |
| 5 | **System Prompt Design** | 元技能，定义 agent 行为边界 | 原 guide 无专门页面，但这是 agent 开发第一步 |

选择标准：
- Agent 开发最常用
- 覆盖不同维度（推理、行动、输出、编排、元设计）
- 升维空间大（原素材不够用，我们能加很多价值）

## 5. 页面 Schema

设计原则：**LLM 可发现性优先**。

- frontmatter 信息密集 → LLM 用 grep/glob 快速定位
- 一句话摘要精准 → 读第一行就能判断相关性
- heading 结构统一 → LLM 可直接跳到需要的 section
- 大白话优先 → 避免术语堆砌

### 完整 Schema

```markdown
---
pattern: <kebab-case-name>
category: <reasoning|action|output|orchestration|meta>
tags: [tag1, tag2, tag3]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: <beginner|intermediate|advanced>
related_wiki: [wiki/prompt-system, wiki/query-loop]
related_patterns: [react, few-shot]
---

# <Pattern 名称>

> 一句话定义：解决什么问题，核心机制是什么。

## 本质

用 2-3 句大白话解释这个 pattern 到底在干嘛。
不堆术语，先建立直觉。

## 什么时候用

- **场景 A** — 描述 + 为什么适合
- **场景 B** — 描述 + 为什么适合
- **Agent 开发场景** — 在 agent 中的典型用法

## 什么时候不用

- **反场景 A** — 为什么不适合 + 会出什么问题
- **反场景 B** — 用了反而有害的情况

## 模板

### 基础版

（可直接复制，带 {{placeholder}} 标记可替换部分）

### Agent 集成版

（嵌入 system prompt / tool definition 的写法）

### 变体

（同一 pattern 的不同形式，说明各自适用条件）

## 组合与选择

- 与 [[cookbook/prompts/patterns/X]] 组合 → 效果 + 场景
- vs [[cookbook/prompts/patterns/Y]] → 什么时候选这个而不是那个

## 模型差异

| 模型 | 表现 | 注意事项 |
|------|------|---------|
| Claude 4 | ... | ... |
| GPT-4 | ... | ... |
| Gemini 2 | ... | ... |
| 开源模型 | ... | ... |

## 常见踩坑

1. **坑名** — 现象 + 原因 + 怎么避
2. **坑名** — 现象 + 原因 + 怎么避

## 来源

- 原始论文：[Author et al. (Year)](url)
- 原始素材：Prompt Engineering Guide / techniques/xxx
- 关联 wiki：[[wiki/prompt-system]]、[[wiki/query-loop]]
```

### Schema 设计决策说明

| 决策 | 理由 |
|------|------|
| frontmatter 用 category 而非自由分类 | 5 个固定类别，LLM 可枚举搜索 |
| related_wiki / related_patterns 放 frontmatter | 机器可解析，支持自动化关联检查 |
| "什么时候用/不用" 作为必填 section | 这是升维的核心价值，原素材最缺的部分 |
| "模板" 分基础版和 Agent 集成版 | 服务两种场景：学习理解 vs 直接开发 |
| "组合与选择" 独立成 section | 打破 pattern 孤岛，建立选择网络 |
| 模型差异用表格 | 结构化，LLM 可按模型名检索 |

## 6. 目录结构

```
cookbook/
├── _index.md                      ← cookbook 总览（板块导航 + 使用指南）
└── prompts/
    ├── _index.md                  ← prompts 板块导航
    ├── patterns/                  ← 通用 pattern（MVP 范围）
    │   ├── chain-of-thought.md
    │   ├── react.md
    │   ├── structured-output.md
    │   ├── prompt-chaining.md
    │   └── system-prompt-design.md
    ├── templates/                 ← 场景模板（MVP 后扩展）
    │   └── _index.md
    └── anti-patterns/             ← 反模式（MVP 后扩展）
        └── _index.md
```

### _index.md 的作用

每个目录的 _index.md 是该板块的"目录页"，包含：
- 板块定位（一句话）
- 所有子页面列表（pattern 名 + 一句话摘要）
- 分类导航（按 category 分组）

这是 skill 导航 LLM 的第一跳——先读 _index.md 确定方向，再读具体页面。

## 7. 与 wiki 的关系

| 维度 | wiki/ | cookbook/ |
|------|-------|----------|
| 回答 | 为什么这样设计 | 怎么动手做 |
| 粒度 | 概念级（如 "Prompt System"） | 技巧级（如 "Chain-of-Thought"） |
| 来源 | 多项目源码对比 | 实践提炼 + 学术锚定 |
| 链接方向 | cookbook pattern → wiki concept | wiki concept → cookbook patterns |

关联方式：
- cookbook 页面的 frontmatter `related_wiki` 指向 wiki 概念页
- wiki 概念页底部加 "实操参考" section 反向链接到 cookbook
- 两边独立演化，通过 wikilink 松耦合

## 8. 验证标准

MVP 完成后，用以下标准判断 schema 是否通过：

### 对 LLM 消费

- [ ] 给 LLM 一个 agent 开发任务，它能通过 _index.md → 具体页面的路径找到需要的 pattern
- [ ] LLM 读完一个 pattern 页面后，能正确使用模板写出可工作的 prompt
- [ ] frontmatter 的 category + tags 能支持 grep 快速过滤

### 对内容质量

- [ ] 每个 pattern 比 Prompt Engineering Guide 的对应页面多了"什么时候用/不用"的决策指导
- [ ] 模板可直接复制使用，带明确的 placeholder
- [ ] "组合与选择" section 建立了 pattern 间的决策网络

### 对人类阅读

- [ ] 非 AI 工程师读完"本质"section 能理解这个 pattern 在干嘛
- [ ] 有经验的工程师能从"模型差异"和"常见踩坑"中获得新信息

## 9. MVP 后扩展路径

```
Phase 1 (当前): cookbook/prompts/patterns/ — 5 个核心 pattern
Phase 2: cookbook/prompts/patterns/ — 铺满剩余 15+ pattern
Phase 3: cookbook/prompts/templates/ — 场景化模板
Phase 4: cookbook/prompts/anti-patterns/ — 反模式
Phase 5: cookbook/tools/ — Tool 定义和编排
Phase 6: cookbook/skills/ — Skill 设计 pattern
Phase 7: skill 开发 — 编写导航 skill，接入 Claude Code
```

## 10. 非目标（明确排除）

- 不做 Prompt Engineering Guide 的中文翻译
- 不做完整的提示词教程（不解释"什么是 prompt"）
- 不做特定框架的使用手册（如 LangChain 教程）
- MVP 阶段不建 skill，先验证内容形态
