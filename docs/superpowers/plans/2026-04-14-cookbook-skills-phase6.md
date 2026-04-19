# Cookbook Skills Phase 6 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建 `cookbook/skills/` 板块，通过分析社区成熟 skill 案例，提炼 5 种 skill 类型的 patterns、templates 和 anti-patterns，服务于 Agent 开发百科全书。

**Architecture:** 纯内容生成任务。Task 1 创建 scaffold，Tasks 2-6 并行写内容页（各 agent 独立读取对应 raw source），Task 7 汇总更新所有 _index.md。

**Tech Stack:** Markdown, YAML frontmatter, 本地 skill 文件作为 raw source

---

## Raw Source 路径（所有 agent 必须知道）

| Source | 路径 | 类型 |
|--------|------|------|
| superpowers skills | `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/` | 纯文本、多文档导航、agent-dispatch |
| gh-cli | `/Users/neo/.claude/skills/gh-cli/SKILL.md` | 外部 CLI |
| deploy-to-vercel | `/Users/neo/.claude/skills/deploy-to-vercel/SKILL.md` | 外部 CLI |
| self-improvement | `/Users/neo/.claude/skills/self-improvement/` | 脚本/Hook |
| webapp-testing | `/Users/neo/.claude/skills/webapp-testing/SKILL.md` | 脚本调用 |

## 目标文件结构

```
cookbook/skills/
├── _index.md
├── patterns/
│   ├── _index.md
│   ├── pure-text-skill.md
│   ├── multi-doc-navigation.md
│   ├── agent-dispatch.md
│   ├── cli-integration.md
│   └── hook-triggered.md
├── templates/
│   ├── _index.md
│   ├── knowledge-skill.md
│   ├── workflow-skill.md
│   └── tool-integration-skill.md
└── anti-patterns/
    ├── _index.md
    ├── skill-as-content-container.md
    ├── missing-trigger-description.md
    └── reinventing-existing-skills.md
```

## Page Schema（所有 pattern 页必须遵守）

```markdown
---
pattern: <kebab-case-name>
category: <pure-text|multi-doc|agent-dispatch|cli-integration|hook-triggered>
tags: [tag1, tag2]
difficulty: <beginner|intermediate|advanced>
examples: [skill-name-1, skill-name-2]
related_patterns: [other-pattern]
---

# <Pattern 名称>

> 一句话定义：解决什么问题，核心机制是什么。

## 本质

2-3 句大白话，先建立直觉。

## 什么时候用

- **场景 A** — 描述 + 为什么适合

## 什么时候不用

- **反场景** — 为什么不适合

## 目录结构

\`\`\`
skill-name/
├── SKILL.md          ← 入口（必须）
└── ...               ← 可选附属文件
\`\`\`

## 关键文件模板

### SKILL.md 基础结构

（带 {{placeholder}} 标记，可直接复制）

### 变体

（同类型不同形式）

## 真实案例

从具体 skill 中提取的结构分析，说明为什么这样设计。

## 常见踩坑

1. **坑名** — 现象 + 原因 + 怎么避

## 来源

- 参考 skill：[skill-name](路径)
```

## Anti-pattern 页 Schema

```markdown
---
anti-pattern: <kebab-case>
category: skill-design
tags: [...]
severity: <high|medium|low>
---

# <Anti-pattern 名称>

> 一句话：这个错误模式是什么。

## 症状

## 错误示例

## 为什么有害

## 正确做法

## 修复检查清单
```

---

## Task 1：Scaffold（必须先完成，其他 Task 依赖此结构）

**Files:**
- Create: `cookbook/skills/_index.md`
- Create: `cookbook/skills/patterns/_index.md`（占位，Task 7 更新）
- Create: `cookbook/skills/templates/_index.md`（占位）
- Create: `cookbook/skills/anti-patterns/_index.md`（占位）

- [ ] **Step 1: 创建目录结构**

```bash
mkdir -p "/Users/neo/Desktop/project/git/ai- knowledge/cookbook/skills/patterns"
mkdir -p "/Users/neo/Desktop/project/git/ai- knowledge/cookbook/skills/templates"
mkdir -p "/Users/neo/Desktop/project/git/ai- knowledge/cookbook/skills/anti-patterns"
```

- [ ] **Step 2: 写 cookbook/skills/_index.md**

内容要求：
- 板块定位（一句话）：Skill 设计的实操参考，覆盖 5 种类型的结构模板和案例
- **"先搜索后创建"原则**：明确说明创建 skill 前必须先用 `find-skills` 或 `npx skills find <query>` 搜索现有 skill
- 三个子板块的链接和一句话描述（patterns / templates / anti-patterns）
- 5 种 skill 类型速览表

- [ ] **Step 3: 创建三个占位 _index.md**

每个只写标题和"内容待更新"即可，Task 7 会填充。

- [ ] **Step 4: 验证目录存在**

```bash
ls "/Users/neo/Desktop/project/git/ai- knowledge/cookbook/skills/"
ls "/Users/neo/Desktop/project/git/ai- knowledge/cookbook/skills/patterns/"
```

---

## Task 2：纯文本型 + 多文档导航型 Pattern（Task 1 完成后并行执行）

**Files:**
- Create: `cookbook/skills/patterns/pure-text-skill.md`
- Create: `cookbook/skills/patterns/multi-doc-navigation.md`

**Raw source（必须先读）:**
- `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/brainstorming/SKILL.md`
- `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/dispatching-parallel-agents/SKILL.md`
- `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/writing-skills/SKILL.md`（多文档型的 meta 案例）
- `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/requesting-code-review/SKILL.md`

- [ ] **Step 1: 读 raw source（4个文件）**

- [ ] **Step 2: 写 pure-text-skill.md**

提炼要点：
- 本质：一个 SKILL.md 文件，纯知识/规则/流程，无脚本无子文件
- 什么时候用：知识型（编码规范、设计原则）、流程型（TDD、code review 流程）
- 什么时候不用：需要执行命令、需要子文档导航
- 模板：包含 frontmatter（name, description）+ 正文结构
- 案例分析：以 `dispatching-parallel-agents` 为例，说明为什么用 Graphviz dot diagram 而非纯文字

- [ ] **Step 3: 写 multi-doc-navigation.md**

提炼要点：
- 本质：SKILL.md 是入口，通过 Read 工具按需加载子文档，适合内容量大的 skill
- 什么时候用：步骤复杂（>200行）、有多条执行路径、子文档可复用
- 关键设计：SKILL.md 必须是"地图"而非"全部内容"，Quick Reference 表让 LLM 知道去哪读哪个文件
- 模板：SKILL.md 入口结构 + 子文档路径约定
- 案例分析：以 `writing-skills`（含 anthropic-best-practices.md、examples/）为例

- [ ] **Step 4: 验证两个文件已创建**

---

## Task 3：Agent Dispatch 型 Pattern（Task 1 完成后并行执行）

**Files:**
- Create: `cookbook/skills/patterns/agent-dispatch.md`

**Raw source（必须先读）:**
- `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/subagent-driven-development/SKILL.md`
- `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/subagent-driven-development/implementer-prompt.md`
- `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/subagent-driven-development/spec-reviewer-prompt.md`
- `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/dispatching-parallel-agents/SKILL.md`

- [ ] **Step 1: 读 raw source（4个文件）**

- [ ] **Step 2: 写 agent-dispatch.md**

提炼要点：
- 本质：skill 的核心是描述"如何构造 agent prompt"和"什么时候派发"，实际执行靠 Agent tool
- 两种子模式：
  1. 单 agent 派发（一个任务一个 agent）
  2. 并行多 agent 派发（多个独立任务并行）
- 关键设计：agent prompt 模板必须是 skill 内容的核心部分（不能只说"派发一个 agent"而没有 prompt 内容）
- 模板：SKILL.md 结构 + agent-prompt 子文件约定
- 案例分析：`subagent-driven-development` 的三文件结构（SKILL.md + implementer + reviewer）

- [ ] **Step 3: 验证文件创建**

---

## Task 4：外部 CLI 型 + 脚本/Hook 型 Pattern（Task 1 完成后并行执行）

**Files:**
- Create: `cookbook/skills/patterns/cli-integration.md`
- Create: `cookbook/skills/patterns/hook-triggered.md`

**Raw source（必须先读）:**
- `/Users/neo/.claude/skills/gh-cli/SKILL.md`（前100行）
- `/Users/neo/.claude/skills/deploy-to-vercel/SKILL.md`（前100行）
- `/Users/neo/.claude/skills/self-improvement/SKILL.md`
- `/Users/neo/.claude/skills/self-improvement/scripts/activator.sh`
- `/Users/neo/.claude/skills/self-improvement/hooks/`目录内容

- [ ] **Step 1: 读 raw source（5个文件）**

```bash
ls /Users/neo/.claude/skills/self-improvement/hooks/
```

- [ ] **Step 2: 写 cli-integration.md**

提炼要点：
- 本质：skill 教 LLM 如何使用特定 CLI 工具，内容=命令参考+决策树+最佳实践
- 与纯文本型的区别：必须包含具体 bash 命令，LLM 需要知道"先检查 X 再执行 Y"
- 关键设计：
  - Prerequisites（安装/认证）必须是第一节
  - 决策树（用什么命令）比命令列表更有用
  - 常见错误的诊断命令
- 模板：prerequisites + 决策树 + 核心命令参考 + 错误处理
- 案例对比：`gh-cli`（全量参考型） vs `deploy-to-vercel`（决策驱动型），说明两种风格的适用场景

- [ ] **Step 3: 写 hook-triggered.md**

提炼要点：
- 本质：skill 带有 shell 脚本，在特定事件（session start/stop、tool call）时自动触发，向 LLM 上下文注入提示
- 适用场景：需要持续提醒（如"每次完成任务后检查学习点"）、需要自动化检测
- 目录结构：scripts/（触发脚本）+ hooks/（Claude Code hooks 配置）
- 关键设计：hook 脚本输出要极简（50-100 tokens），避免污染 context
- 模板：SKILL.md + scripts/activator.sh 结构
- 案例分析：`self-improvement` 的 activator.sh → 输出 `<self-improvement-reminder>` XML 标签

- [ ] **Step 4: 验证两个文件已创建**

---

## Task 5：Templates（Task 1 完成后并行执行）

**Files:**
- Create: `cookbook/skills/templates/knowledge-skill.md`
- Create: `cookbook/skills/templates/workflow-skill.md`
- Create: `cookbook/skills/templates/tool-integration-skill.md`

不需要读新 raw source，基于前面已提炼的 pattern 知识写模板。

- [ ] **Step 1: 写 knowledge-skill.md**

适用：编码规范、设计原则、概念解释类 skill
内容：完整可复制的 SKILL.md 模板，带 {{placeholder}}

```markdown
---
name: {{skill-name}}
description: "{{一句话描述触发时机和用途}}"
---

# {{Skill 标题}}

## Overview

{{2-3句：这个skill解决什么问题，核心价值是什么}}

## When to Use

- {{场景1}}
- {{场景2}}

## {{核心内容章节1}}

{{内容}}

## {{核心内容章节2}}

{{内容}}

## Common Mistakes

| 错误 | 后果 | 正确做法 |
|------|------|----------|
| {{错误}} | {{后果}} | {{正确}} |
```

- [ ] **Step 2: 写 workflow-skill.md**

适用：有明确执行步骤的流程型 skill（如 code review、debugging、deployment）
内容：SKILL.md 入口模板 + 可选子文档结构

特别说明：
- description 字段必须包含明确的触发条件（"Use when..."）
- 流程用 dot diagram 而非 ordered list，更利于 LLM 理解分支
- 步骤超过 5 个就考虑拆子文档

- [ ] **Step 3: 写 tool-integration-skill.md**

适用：封装 CLI 工具或外部 API 的 skill
内容：带 prerequisites + 决策树 + 命令参考的完整模板

- [ ] **Step 4: 验证三个文件创建**

---

## Task 6：Anti-patterns（Task 1 完成后并行执行）

**Files:**
- Create: `cookbook/skills/anti-patterns/skill-as-content-container.md`
- Create: `cookbook/skills/anti-patterns/missing-trigger-description.md`
- Create: `cookbook/skills/anti-patterns/reinventing-existing-skills.md`

- [ ] **Step 1: 写 skill-as-content-container.md**

症状：skill 把所有知识都塞进 SKILL.md（>500行），LLM 每次都要加载全部内容
错误示例：一个 SKILL.md 同时包含 3 种语言的编码规范
为什么有害：context 污染、加载慢、LLM 注意力稀释
正确做法：SKILL.md 是入口地图，具体内容放子文档，按需 Read

- [ ] **Step 2: 写 missing-trigger-description.md**

症状：description 字段写的是 skill 能做什么，而非什么时候用
错误示例：`description: "Contains best practices for React development"`
为什么有害：LLM 无法判断何时应该触发这个 skill，导致该用不用
正确做法：`description: "Use when building React components, reviewing React code, or asking about React patterns"`
规则：description 必须以 "Use when..." 开头，列出 2-3 个具体触发场景

- [ ] **Step 3: 写 reinventing-existing-skills.md**

症状：创建了一个社区已有的成熟 skill（如自己写 git commit 规范，但 github/awesome-copilot@git-commit 已有 23.9K 安装）
为什么有害：维护负担、质量不如社区积累的版本、浪费时间
正确做法：创建新 skill 前先执行 `npx skills find <query>` 搜索，找到再评估是否直接用或基于它改
修复检查清单：
- [ ] 已用 `npx skills find` 搜索相关关键词？
- [ ] 已浏览 skills.sh 确认无重复？
- [ ] 确认现有 skill 无法满足需求（太通用/语言不对/场景不匹配）？

- [ ] **Step 4: 验证三个文件创建**

---

## Task 7：更新所有 _index.md（Tasks 2-6 全部完成后执行）

**Files:**
- Modify: `cookbook/skills/_index.md`（补充完整内容）
- Modify: `cookbook/skills/patterns/_index.md`（列出 5 个 pattern）
- Modify: `cookbook/skills/templates/_index.md`（列出 3 个 template）
- Modify: `cookbook/skills/anti-patterns/_index.md`（列出 3 个 anti-pattern）
- Modify: `cookbook/_index.md`（添加 skills 板块入口）
- Modify: `cookbook/tools/_index.md`（检查是否需要添加 skill-system 交叉引用）

- [ ] **Step 1: 读现有 _index 文件**

```bash
cat "/Users/neo/Desktop/project/git/ai- knowledge/cookbook/_index.md"
cat "/Users/neo/Desktop/project/git/ai- knowledge/cookbook/skills/_index.md"
```

- [ ] **Step 2: 更新 cookbook/skills/patterns/_index.md**

格式：每个 pattern 一行，包含名称、一句话摘要、适用场景标签

- [ ] **Step 3: 更新 cookbook/skills/templates/_index.md 和 anti-patterns/_index.md**

- [ ] **Step 4: 更新 cookbook/skills/_index.md**

完整内容版本，包含：
- 板块定位
- **"先搜索后创建"原则**（重要！）
- 三个子板块的导航
- 5种类型速览表（带链接）

- [ ] **Step 5: 更新顶层 cookbook/_index.md**

在现有 prompts/ 和 tools/ 后面添加 skills/ 板块入口。

- [ ] **Step 6: git commit**

```bash
cd "/Users/neo/Desktop/project/git/ai- knowledge"
git add cookbook/skills/
git add cookbook/_index.md
git commit -m "feat(cookbook): add skills phase 6 - 5 patterns + 3 templates + 3 anti-patterns"
```

---

## 执行顺序

```
Task 1 (Scaffold)
    ↓ 完成后
Task 2 + Task 3 + Task 4 + Task 5 + Task 6  ← 5个并行
    ↓ 全部完成后
Task 7 (Index + commit)
```

Tasks 2-6 互相独立，必须并行派发。
