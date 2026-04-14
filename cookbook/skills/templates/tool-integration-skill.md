---
template: tool-integration-skill
category: template
tags: [tool, cli, external-service, integration]
use-for: 封装 CLI 工具、外部 API、云服务操作
---

# 工具集成型 Skill 模板

> 适用于：封装 CLI 命令或外部服务调用，需要状态检查、错误处理、标准化命令序列的场景。

## 使用说明

1. 复制下方模板到 `~/.claude/skills/<skill-name>/SKILL.md`
2. 替换所有 `{{placeholder}}`
3. Prerequisites 节必须包含完整的安装 + 认证 + 验证三步，缺一不可
4. 每条命令后标注：是否幂等、是否可回滚、是否需要用户确认

## 适用场景

- **CLI 封装** — git、docker、kubectl、vercel、gh 等命令的标准化用法
- **云服务操作** — AWS/GCP/Azure 资源管理、数据库操作
- **外部 API** — 调用第三方服务、webhook 触发、数据同步
- **本地工具** — 构建工具、测试运行器、代码生成器

---

## 模板（复制以下全部内容）

```markdown
---
name: {{SKILL_NAME}}
description: "{{WHAT_THE_TOOL_DOES}} via {{TOOL_NAME}}. Use when user asks to {{ACTION_1}}, {{ACTION_2}}, or {{ACTION_3}}."
---

# {{SKILL_TITLE}}

## Overview

{{ONE_SENTENCE_WHAT_THIS_SKILL_ENABLES}}. Always check current state before executing commands — do not assume initial state.

## When to Use

- Use when user asks to {{ACTION_SCENARIO_1}}
- Use when user asks to {{ACTION_SCENARIO_2}}
- Use when {{SYSTEM_STATE_THAT_TRIGGERS_THIS}}
- Do NOT use when {{ANTI_SCENARIO}} — {{ALTERNATIVE_APPROACH}} is better

## Prerequisites

### 1. Installation

```bash
# Check if installed
{{TOOL_NAME}} --version 2>/dev/null || echo "not installed"

# Install if missing
{{INSTALL_COMMAND}}
```

### 2. Authentication

```bash
# Check authentication status
{{AUTH_CHECK_COMMAND}}

# Authenticate if needed
{{AUTH_COMMAND}}
```

### 3. Verify Setup

```bash
{{VERIFY_COMMAND}}
```

Expected output: `{{EXPECTED_VERIFY_OUTPUT}}`

If verification fails: {{TROUBLESHOOT_HINT}}

## Decision Tree: Choose the Right Command

```dot
digraph {{SKILL_NAME_NOSPACE}}_decision {
    "Check current state" [shape=box];
    "{{STATE_CHECK_QUESTION}}" [shape=diamond];
    "{{BRANCH_A_LABEL}}" [shape=diamond];
    "{{BRANCH_B_LABEL}}" [shape=diamond];
    "{{COMMAND_A}}" [shape=box];
    "{{COMMAND_B}}" [shape=box];
    "{{COMMAND_C}}" [shape=box];
    "Verify result" [shape=box];
    "Success" [shape=doublecircle];
    "Handle error" [shape=box];

    "Check current state" -> "{{STATE_CHECK_QUESTION}}";
    "{{STATE_CHECK_QUESTION}}" -> "{{BRANCH_A_LABEL}}" [label="{{STATE_A}}"];
    "{{STATE_CHECK_QUESTION}}" -> "{{BRANCH_B_LABEL}}" [label="{{STATE_B}}"];
    "{{BRANCH_A_LABEL}}" -> "{{COMMAND_A}}" [label="{{CONDITION_A1}}"];
    "{{BRANCH_A_LABEL}}" -> "{{COMMAND_B}}" [label="{{CONDITION_A2}}"];
    "{{BRANCH_B_LABEL}}" -> "{{COMMAND_C}}";
    "{{COMMAND_A}}" -> "Verify result";
    "{{COMMAND_B}}" -> "Verify result";
    "{{COMMAND_C}}" -> "Verify result";
    "Verify result" -> "Success" [label="ok"];
    "Verify result" -> "Handle error" [label="failed"];
    "Handle error" -> "Check current state" [label="retry"];
}
```

## Core Commands

### {{COMMAND_GROUP_1}}

```bash
# {{COMMAND_1_DESCRIPTION}}
{{COMMAND_1}}

# {{COMMAND_2_DESCRIPTION}}
{{COMMAND_2}}
```

| Flag | Effect | When to Use |
|---|---|---|
| `{{FLAG_1}}` | {{FLAG_1_EFFECT}} | {{FLAG_1_WHEN}} |
| `{{FLAG_2}}` | {{FLAG_2_EFFECT}} | {{FLAG_2_WHEN}} |
| `{{FLAG_3}}` | {{FLAG_3_EFFECT}} | {{FLAG_3_WHEN}} |

### {{COMMAND_GROUP_2}}

```bash
# {{COMMAND_3_DESCRIPTION}}
{{COMMAND_3}}

# {{COMMAND_4_DESCRIPTION}} — run before {{DESTRUCTIVE_COMMAND}} to preview
{{DRY_RUN_COMMAND}}
```

> Always run `{{DRY_RUN_COMMAND}}` before `{{DESTRUCTIVE_COMMAND}}` unless the user explicitly opts out.

## State Check Commands

Run these before deciding which command to use:

```bash
# {{STATE_CHECK_1_DESCRIPTION}}
{{STATE_CHECK_1_COMMAND}}

# {{STATE_CHECK_2_DESCRIPTION}}
{{STATE_CHECK_2_COMMAND}}

# {{STATE_CHECK_3_DESCRIPTION}}
{{STATE_CHECK_3_COMMAND}}
```

## Error Handling

| Exit Code / Error Message | Likely Cause | Diagnostic Command | Fix |
|---|---|---|---|
| `{{ERROR_1}}` | {{CAUSE_1}} | `{{DIAG_CMD_1}}` | {{FIX_1}} |
| `{{ERROR_2}}` | {{CAUSE_2}} | `{{DIAG_CMD_2}}` | {{FIX_2}} |
| `{{ERROR_3}}` | {{CAUSE_3}} | `{{DIAG_CMD_3}}` | {{FIX_3}} |
| `{{ERROR_4}}` | {{CAUSE_4}} | `{{DIAG_CMD_4}}` | {{FIX_4}} |

## Rollback

If the operation needs to be reversed:

```bash
# {{ROLLBACK_DESCRIPTION}}
{{ROLLBACK_COMMAND}}

# Verify rollback succeeded
{{ROLLBACK_VERIFY_COMMAND}}
```

> Note: {{ROLLBACK_LIMITATION}} — this cannot be undone after {{POINT_OF_NO_RETURN}}.
```

---

## 关键设计原则

### Prerequisites 三步必须齐全

工具集成 skill 失败的最常见原因不是命令错了，而是工具没安装或认证过期。三步缺一不可：

1. **Installation** — 先检查（`tool --version`），再给安装命令。不能假设工具已安装。
2. **Authentication** — 给出状态检查命令（`whoami`、`auth status`），再给认证命令。不能假设已登录。
3. **Verify Setup** — 给一个轻量的端到端验证命令，明确期望输出和失败时的提示。

### 先查状态，再选命令

工具集成的核心模式是"先查，再做"。State Check Commands 节提供的命令用于了解当前状态，Decision Tree 节说明根据状态选哪个命令分支。这防止了在已完成的步骤上重复操作，或在错误的初始状态下执行命令。

### 决策树而非条件文字

工具调用中分支逻辑最容易被 LLM 误读——文字描述的条件容易跳步骤或走错分支。dot diagram 将条件显式编码为边上的 label，强制 LLM 按图遍历。

### 错误处理表：四列缺一不可

| 字段 | 作用 |
|---|---|
| Exit Code / Error Message | 识别——这是什么错误 |
| Likely Cause | 理解——为什么会发生 |
| Diagnostic Command | 诊断——运行什么来确认原因 |
| Fix | 修复——怎么解决 |

只给错误信息不给诊断命令，LLM 只会猜测原因而不是实际诊断。

### Dry-run 优先原则

凡是有 dry-run 或 preview 选项的破坏性命令，必须在 Core Commands 节明确标注，并在对应操作前自动执行 dry-run，除非用户明确要求跳过。

### Rollback 节：声明不可逆点

告诉 LLM 哪些操作可以回滚、如何回滚，以及不可逆的时间点在哪里。这防止 LLM 在执行破坏性操作后才意识到无法撤销。

### description 字段：列出动词短语

工具类 skill 的触发词是动词短语（deploy、push、create、delete、sync）而非状态描述。description 应枚举用户会说的具体动词：

```yaml
description: "Manages Docker containers and images via docker CLI.
  Use when user asks to build an image, run a container, stop or remove containers,
  push to a registry, or debug container logs."
```
