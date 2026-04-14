---
pattern: multi-doc-navigation
category: multi-doc
tags: [navigation, sub-documents, large-skill, index]
difficulty: intermediate
examples: [writing-skills, requesting-code-review, create-agent-skill]
related_patterns: [pure-text-skill, agent-dispatch]
---

# 多文档导航型 Skill

> 一句话定义：SKILL.md 是地图，不是全部内容——它告诉 LLM 去哪个子文件读什么，而不是把所有内容塞进一个文件。

## 本质

当 skill 的内容超过单文件能清晰呈现的范围时，强行塞进一个 SKILL.md 会带来两个问题：

1. **Context 浪费**：LLM 必须一次性加载所有内容，但大多数情况只需要其中一部分。
2. **导航困难**：LLM 在超长文件里难以定位，注意力在阅读过程中逐渐分散。

多文档导航模式的核心洞察：**SKILL.md 的职责是让 LLM 在第一屏就知道去哪读什么**。真正的内容放在有语义命名的子文件里，按需加载。

这不是文档组织偏好，是 context 效率设计。一个 LLM 读完 SKILL.md 的入口表格，就知道自己要的信息在哪个文件，然后只读那个文件——节省了加载整个知识库的代价。

## 什么时候用

- **内容自然分成多个独立主题** — 每个子主题有独立的使用场景，LLM 通常只需要其中一个。例如：writing-skills 里，"如何测试 skill"（testing-skills-with-subagents.md）和"Anthropic 官方最佳实践"（anthropic-best-practices.md）是独立的参考资料，不需要同时加载。
- **包含可复用文件（脚本、模板、dot 格式约定）** — 这类文件本来就应该独立存放，以便被其他 skill 引用或直接执行。
- **存在"重量级参考资料"（100+ 行）** — API 文档、大型代码模板、详细 checklist。内联会拖慢每次加载，独立成文件按需读取更高效。
- **有明确的"主流程"和"详情"之分** — 主流程放 SKILL.md，详情放子文件。这对应 "progressive disclosure"（渐进式披露）的信息架构原则。

## 什么时候不用

- **内容总量 < 500 词** — 拆文件反而增加导航摩擦。LLM 需要读 SKILL.md、判断去哪、再读子文件，不如一次读完。改用 pure-text-skill。
- **子文件之间有强依赖** — 如果 A 文件不看就无法理解 B 文件，拆分反而制造认知断层。应合并或重新设计依赖关系。
- **内容不会复用** — 如果一个子文件只会被这一个 SKILL.md 引用，且内容很短，内联更简洁。

## 目录结构

```
skill-name/
├── SKILL.md              ← 入口导航（地图）
├── topic-a.md            ← 子文档：主题 A
├── topic-b.md            ← 子文档：主题 B
├── example.ts            ← 可复用脚本（如果有）
└── conventions.dot       ← 可复用配置（如果有）
```

**子文档命名约定**：动词-名词结构（`writing-memory.md`、`testing-skills-with-subagents.md`），而不是 `doc1.md`、`part2.md`。命名本身就是语义索引。

## SKILL.md 模板

SKILL.md 的核心是一张 Quick Reference 表，LLM 读第一屏就能定位需要的子文件：

```markdown
---
name: {{skill-name}}
description: Use when {{triggering conditions only, no workflow summary}}
---

# {{Skill Human Name}}

## Overview

{{Core principle in 1-2 sentences.}}

**REQUIRED BACKGROUND:** You MUST understand {{dependency-skill}} before using this skill.

**Official guidance:** For {{topic}}, see {{sub-document-name.md}}. This document provides additional patterns.

## Quick Reference

| Task | Where to look |
|------|--------------|
| {{Task A}} | See {{file-a.md}} |
| {{Task B}} | See {{file-b.md}} |
| {{Task C, inline here}} | [Section below](#{{anchor}}) |
| {{Task D}} | See {{file-d.md}} or inline {{tool-name --help}} |

## {{Main Concept or Core Pattern}}

{{The essential content that belongs inline — the "always need" part.
Keep this section to the minimum required for understanding.}}

## When to {{Do X}}

{{Criteria for the most common decisions. Can include small flowchart if non-obvious.}}

**Create when:**
- {{Condition 1}}
- {{Condition 2}}

**Don't create for:**
- {{Anti-condition 1}}

## {{Process Steps (if applicable)}}

### 1. {{Step One}}

{{Description. If details are in a sub-document, reference it explicitly:}}

**Details:** See {{sub-document.md}} for complete {{methodology/reference/examples}}.

### 2. {{Step Two}}

{{Description.}}

## Common Mistakes

{{Anti-patterns and fixes.}}

## Sub-Documents

{{Optional: brief description of what each sub-file contains, for cases where
the Quick Reference table isn't self-explanatory.}}

- **{{file-a.md}}** — {{one line description}}
- **{{file-b.md}}** — {{one line description}}
```

关键约束：
- Quick Reference 表必须在 **前 30 行以内** 出现，确保 LLM 在"第一屏"就看到导航
- 子文档引用使用文件名（`testing-skills-with-subagents.md`），不使用 `@` 前缀
- `@` 语法会强制预加载文件，消耗 200k+ context，只在绝对必须时使用
- REQUIRED BACKGROUND 使用 skill 名称（`superpowers:test-driven-development`），不使用文件路径

## 子文档引用方式

多文档导航 skill 里有两种合法的子文档引用方式：

**方式 A：说明性引用**（推荐）
```markdown
See testing-skills-with-subagents.md for the complete testing methodology.
```
LLM 在需要时主动读取，不强制加载。

**方式 B：REQUIRED BACKGROUND**
```markdown
**REQUIRED BACKGROUND:** You MUST understand superpowers:test-driven-development before using this skill.
```
用于"不看就无法理解主 skill"的强依赖，用 skill 名称而非文件路径。

**方式 C：@-syntax**（慎用）
```markdown
See @graphviz-conventions.dot for graphviz style rules.
```
只用于确实需要立即加载的内容（如当前任务必须用到的模板），否则浪费 context。

## 真实案例分析

### writing-skills：入口文档如何做到"够用但不冗余"

writing-skills 的目录结构：
```
writing-skills/
├── SKILL.md                          ← 核心流程 + 导航
├── anthropic-best-practices.md       ← 官方权威指南（重量级）
├── testing-skills-with-subagents.md  ← 完整测试方法论（重量级）
├── persuasion-principles.md          ← 研究基础（可选背景）
├── graphviz-conventions.dot          ← 可复用格式约定
├── render-graphs.js                  ← 可执行脚本
└── examples/
    └── CLAUDE_MD_TESTING.md          ← 测试用例示例
```

SKILL.md 的设计策略：

1. **主体放流程和原则**：TDD 映射表、skill 类型定义、frontmatter 规范、CSO 章节——这些是每次使用 writing-skills 都需要的核心知识，内联合理。

2. **重量级内容用说明性引用**：
   ```markdown
   **Official guidance:** For Anthropic's official skill authoring best practices,
   see anthropic-best-practices.md.
   ```
   anthropic-best-practices.md 超过 10000 tokens，每次都加载是浪费。需要时 LLM 主动读取。

3. **测试方法论同样处理**：
   ```markdown
   **Testing methodology:** See @testing-skills-with-subagents.md for the complete
   testing methodology
   ```
   注意这里用了 `@`——写技术文档本身就需要知道完整测试方法，所以这里的强制加载是有意为之的。

4. **子文件命名即语义**：`testing-skills-with-subagents.md` vs `testing.md`。前者一眼看出"用 subagent 来测试 skill"，后者只知道"跟测试有关"。命名让 LLM 在不读内容的情况下也能判断相关性。

### requesting-code-review：最小多文档设计

requesting-code-review 只有一个子文件：

```
requesting-code-review/
├── SKILL.md           ← 完整流程 + 操作指南（短，内联）
└── code-reviewer.md  ← Agent prompt 模板（可复用）
```

这是"刚好需要拆一个文件"的最小形态。SKILL.md 末尾一行引用：
```markdown
See template at: requesting-code-review/code-reviewer.md
```

code-reviewer.md 独立存放的原因不是"内容太长"，而是**它本身是一个可直接使用的 agent prompt 模板**——`{WHAT_WAS_IMPLEMENTED}`、`{BASE_SHA}` 等 placeholder 让它既是文档又是可执行的 prompt，独立文件便于 Task tool 直接引用。

这个案例说明：拆文件的理由可以是"文件的性质"（可执行模板），而不只是"内容量"。

### Quick Reference 表的设计价值

Quick Reference 表解决了一个具体问题：**LLM 加载一个 skill 时，不应该被迫读完整个文件才能知道自己需要什么**。

没有 Quick Reference 时，LLM 的阅读路径是：
```
Overview → When to Use → Core Pattern → Implementation → ...（读完）→ 发现第 8 章有需要的内容
```

有 Quick Reference 时：
```
Overview → Quick Reference 表（30行内）→ 发现"需要的内容在 file-b.md"→ 直接读 file-b.md
```

writing-skills 的 Quick Reference 体现在 TDD 映射表（让 LLM 立即理解"这是 TDD for documentation"）和目录结构对比图（三种文件组织模式各自何时用）。这两处内容在文件靠前的位置，确保 LLM 能快速确认"这个 skill 是否是我需要的"。

## 常见踩坑

**SKILL.md 变成内容仓库** — 随着时间推移，往 SKILL.md 里添加越来越多细节，最终单文件超过 600 行，导航作用消失。修复：定期检查 SKILL.md，把"详情"下沉到子文件，保持 SKILL.md 的"地图"定位。

**子文件用序号命名** — `doc1.md`、`part2.md`、`section3.md`。LLM 无法从名称判断内容，导航失效。修复：全部改为动词-名词结构，名称描述内容。

**过度拆分** — 把 200 词的内容拆成 3 个文件，制造无谓的导航跳转。修复：子文件应该有独立的使用场景，不是为了拆而拆。

**Quick Reference 太靠后** — 把导航表放在文件末尾，LLM 需要读完全文才能发现导航。修复：导航表必须在前 30 行出现，最好紧跟 Overview。

**@-syntax 滥用** — 用 `@file.md` 引用所有子文档，每次加载 skill 就把所有子文件全部预加载。修复：只在"当前任务必须立即使用"的情况下用 `@`，其余用说明性引用。

**子文件之间没有独立性** — 文件 B 必须在读完文件 A 之后才能理解。这样的拆分方式破坏了按需加载的价值。修复：每个子文件应该能独立阅读，必要时在文件开头注明前置依赖。

## 来源

- raw: `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/writing-skills/SKILL.md`
- raw: `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/requesting-code-review/SKILL.md`
- raw: `/Users/neo/.claude/plugins/cache/superpowers-marketplace/superpowers/4.3.1/skills/requesting-code-review/code-reviewer.md`
- 相关模式: [[pure-text-skill]], [[agent-dispatch]]
