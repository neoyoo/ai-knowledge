---
template: knowledge-skill
category: template
tags: [knowledge, rules, standards, single-file]
use-for: 编码规范、设计原则、概念参考、最佳实践
---

# 知识/规则型 Skill 模板

> 适用于：只需要告诉 LLM "知道什么"和"遵守什么规则"的场景，无需执行脚本或派发 agent。

## 使用说明

1. 复制下方模板到 `~/.claude/skills/<skill-name>/SKILL.md`
2. 替换所有 `{{placeholder}}`
3. 运行 `npx skills find <query>` 确认没有现成 skill 再写
4. 知识型 skill 通常单文件即可；内容超过 400 行时才考虑拆子文档

## 适用场景

- **编码规范** — 命名约定、格式要求、禁止模式
- **设计原则** — 架构决策、API 设计准则、系统约束
- **概念参考** — 领域术语、分类体系、知识索引
- **最佳实践** — 安全规则、性能指南、代码审查标准

---

## 模板（复制以下全部内容）

```markdown
---
name: {{SKILL_NAME}}
description: "{{WHAT_IT_KNOWS_AND_WHEN_TO_USE_IT}}. Use when {{TRIGGER_CONDITION_1}}, when {{TRIGGER_CONDITION_2}}, or when {{TRIGGER_CONDITION_3}}."
---

# {{SKILL_TITLE}}

## Overview

{{ONE_SENTENCE_DEFINITION}}. {{WHERE_THIS_APPLIES_IN_THE_SYSTEM}}.

## When to Use

- Use when {{SCENARIO_1_CONCRETE_EXAMPLE}}
- Use when {{SCENARIO_2_CONCRETE_EXAMPLE}}
- Use when {{SCENARIO_3_CONCRETE_EXAMPLE}}
- Use when {{SCENARIO_4_CONCRETE_EXAMPLE}}
- Do NOT use when {{ANTI_SCENARIO}} — use {{ALTERNATIVE}} instead

## {{CORE_SECTION_1_NAME}}

{{SECTION_1_CONTENT}}

Key rules:
- {{RULE_1}}
- {{RULE_2}}
- {{RULE_3}}

## {{CORE_SECTION_2_NAME}}

{{SECTION_2_CONTENT}}

| {{COLUMN_A}} | {{COLUMN_B}} | {{COLUMN_C}} |
|---|---|---|
| {{ROW_1_A}} | {{ROW_1_B}} | {{ROW_1_C}} |
| {{ROW_2_A}} | {{ROW_2_B}} | {{ROW_2_C}} |
| {{ROW_3_A}} | {{ROW_3_B}} | {{ROW_3_C}} |

## {{CORE_SECTION_3_NAME}}

{{SECTION_3_CONTENT}}

## Common Mistakes

| Mistake | Why It's Wrong | Correct Approach |
|---|---|---|
| {{MISTAKE_1}} | {{REASON_1}} | {{FIX_1}} |
| {{MISTAKE_2}} | {{REASON_2}} | {{FIX_2}} |
| {{MISTAKE_3}} | {{REASON_3}} | {{FIX_3}} |

## Quick Reference

| {{ITEM}} | {{RULE_OR_VALUE}} |
|---|---|
| {{ITEM_1}} | {{VALUE_1}} |
| {{ITEM_2}} | {{VALUE_2}} |
| {{ITEM_3}} | {{VALUE_3}} |
```

---

## 关键设计原则

### description 字段：发现靠它

`description` 是 Claude 选择 skill 的唯一依据。必须包含两部分：
- **做什么**：skill 覆盖的知识范围
- **何时用**：`Use when ...` 列出 2-3 个具体触发词

差的写法：`"Coding standards for this project"`
好的写法：`"Python coding standards: naming, formatting, imports, type hints. Use when writing Python code, reviewing Python PRs, or when asked about code style."`

### When to Use：具体动词句

每条必须是 `Use when ...` 句型，描述真实场景而非泛泛的"需要时"。

差：`Use when you need coding rules`
好：`Use when writing a new Python module`, `Use when reviewing a PR that touches database models`

### 章节数量：2-3 个核心章节

知识型 skill 不是百科全书。聚焦最常查阅的 2-3 个主题，其余内容按需另建子文档（用 Read 工具加载）。

### Common Mistakes 表格：必有

这是知识型 skill 最有价值的部分——记录的不是规则本身，而是人（或 LLM）最容易犯错的地方。

### Quick Reference 表：可选

仅在有大量需要快速查找的映射关系时添加（如状态码、配置项、命名前缀）。内容稀少时省略。

### 单文件优先

知识型 skill 的内容通常可以放在一个文件里。以下情况才考虑拆子文档：
- 某一节超过 150 行
- 有针对不同角色（如 backend/frontend）的分支内容
- 有需要频繁独立更新的附录（如错误码列表）
