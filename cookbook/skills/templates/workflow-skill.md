---
template: workflow-skill
category: template
tags: [workflow, process, steps, checkpoints]
use-for: code review、debugging、部署流程、有明确步骤的操作序列
---

# 工作流型 Skill 模板

> 适用于：有明确执行步骤、包含决策分支、需要检查点确认的操作流程。

## 使用说明

1. 复制下方模板到 `~/.claude/skills/<skill-name>/SKILL.md`
2. 替换所有 `{{placeholder}}`
3. 步骤数 > 5 个或有多条执行路径时，将详细步骤拆到子文档（如 `steps/`）
4. 用 dot diagram 而非 ordered list 表达流程，原因见下方设计原则

## 适用场景

- **Code Review** — 有标准化检查项和结论输出格式
- **Debugging** — 分层诊断，有明确的升级路径
- **部署/发布** — 前置检查 → 执行 → 验证 → 回滚决策
- **Onboarding** — 多阶段流程，每阶段有验收标准

---

## 模板（复制以下全部内容）

```markdown
---
name: {{SKILL_NAME}}
description: "{{PROCESS_DESCRIPTION}}. Use when user asks to {{TRIGGER_1}}, when {{TRIGGER_2}} fails, when {{TRIGGER_3}}."
---

# {{SKILL_TITLE}}

## Overview

{{ONE_SENTENCE_WHAT_THIS_PROCESS_ACHIEVES}}. Always follow this process — do not skip steps even if the task seems simple.

<HARD-GATE>
Do NOT proceed to {{CRITICAL_STEP}} until {{PREREQUISITE_CONDITION}} is confirmed. This applies regardless of perceived simplicity.
</HARD-GATE>

## When to Use

- Use when user asks to {{TRIGGER_SCENARIO_1}}
- Use when {{SYSTEM_OR_STATE}} shows {{SYMPTOM_THAT_TRIGGERS_THIS}}
- Use when {{TRIGGER_SCENARIO_3}}
- Do NOT use when {{ANTI_SCENARIO}} — use {{ALTERNATIVE_SKILL}} instead

## Process Flow

```dot
digraph {{SKILL_NAME_NOSPACE}} {
    "{{STEP_1}}" [shape=box];
    "{{STEP_2}}" [shape=box];
    "{{DECISION_1}}" [shape=diamond];
    "{{STEP_3A}}" [shape=box];
    "{{STEP_3B}}" [shape=box];
    "{{CHECKPOINT_1}}" [shape=parallelogram];
    "{{STEP_4}}" [shape=box];
    "{{TERMINAL_STATE}}" [shape=doublecircle];

    "{{STEP_1}}" -> "{{STEP_2}}";
    "{{STEP_2}}" -> "{{DECISION_1}}";
    "{{DECISION_1}}" -> "{{STEP_3A}}" [label="{{CONDITION_YES}}"];
    "{{DECISION_1}}" -> "{{STEP_3B}}" [label="{{CONDITION_NO}}"];
    "{{STEP_3A}}" -> "{{CHECKPOINT_1}}";
    "{{STEP_3B}}" -> "{{CHECKPOINT_1}}";
    "{{CHECKPOINT_1}}" -> "{{STEP_4}}" [label="confirmed"];
    "{{CHECKPOINT_1}}" -> "{{STEP_2}}" [label="retry"];
    "{{STEP_4}}" -> "{{TERMINAL_STATE}}";
}
```

The terminal state is **{{TERMINAL_STATE}}**. Do not invoke other skills or take additional actions after reaching it.

## Checklist

Complete each item in order. Do not skip items.

1. **{{STEP_1_NAME}}** — {{STEP_1_DESCRIPTION}}
2. **{{STEP_2_NAME}}** — {{STEP_2_DESCRIPTION}}

   > Checkpoint: {{CHECKPOINT_1_QUESTION}} If no, go back to step 1.

3. **{{STEP_3_NAME}}** — {{STEP_3_DESCRIPTION}}
4. **{{STEP_4_NAME}}** — {{STEP_4_DESCRIPTION}}

   > Checkpoint: {{CHECKPOINT_2_QUESTION}} If uncertain, {{CHECKPOINT_2_ACTION}}.

5. **{{STEP_5_NAME}}** — {{STEP_5_DESCRIPTION}}

## {{DETAILED_SECTION_1}}

{{SECTION_1_CONTENT}}

For complex scenarios: Read `{{SUBFILE_PATH}}` for detailed instructions.

## Output Format

{{DESCRIBE_WHAT_TO_PRODUCE_AT_THE_END}}

Example:
```
{{OUTPUT_EXAMPLE_LINE_1}}
{{OUTPUT_EXAMPLE_LINE_2}}
{{OUTPUT_EXAMPLE_LINE_3}}
```

## Error Recovery

| Error | Likely Cause | Recovery Action |
|---|---|---|
| {{ERROR_1}} | {{CAUSE_1}} | {{RECOVERY_1}} |
| {{ERROR_2}} | {{CAUSE_2}} | {{RECOVERY_2}} |
```

---

## 关键设计原则

### description 字段的三种触发句型

工作流型 skill 有三类典型触发条件，description 应覆盖全部：

```yaml
description: "Reviews code for quality, bugs, and standards compliance.
  Use when user asks to review code or a PR,
  when CI checks fail and need investigation,
  when merging a feature branch."
```

- `Use when user asks to ...` — 用户主动请求
- `when ... fails` — 某个状态/检查失败
- `when ... needs ...` — 某个上下文需要此流程

### dot diagram 而非 ordered list

ordered list 无法表达分支和循环，阅读时也难以追踪当前在哪一步。dot diagram 做到三件事：
1. 显式表达决策节点（`shape=diamond`）
2. 显式标注边的条件（`[label="condition"]`）
3. 显式标注终态（`shape=doublecircle`）——让 LLM 知道何时停止

节点形状约定：
- `box` — 普通步骤
- `diamond` — 决策/分支
- `parallelogram` — 检查点（等待用户确认）
- `doublecircle` — 终态

### HARD-GATE：关键操作前的强制停止

对于不可逆操作（删除、部署、合并），必须在 Overview 后加 `<HARD-GATE>` 块，明确：
- 什么操作需要停止
- 停止条件是什么
- 适用范围（"regardless of perceived simplicity"）

### Checkpoint 放在步骤内，不是步骤间

检查点是步骤的一部分，紧跟在需要验证的步骤后面，用 `> Checkpoint:` 格式标注。包含：
- 验证问题（是/否）
- 不通过时的操作（回到哪一步，或执行什么补救）

### 何时拆子文档

满足以下任一条件时将详细内容拆到子文档：
- 某个步骤的详细说明超过 50 行
- 有多条互斥的执行路径（如不同操作系统、不同项目类型）
- 有需要独立维护的参考内容（如错误码手册、配置样例）

主 SKILL.md 保留流程骨架，用 `Read \`path/to/subfile.md\`` 在需要时加载详情。

### Output Format 节：明确终态长什么样

LLM 很容易在生成一个合理的中间产物后停下来。明确的 Output Format 节告诉它：最终要交付什么格式、包含哪些字段。提供一个真实的 example 比描述更有效。
