---
pattern: hook-triggered
category: hook-triggered
tags: [hook, script, automation, session, event-driven]
difficulty: advanced
examples: [self-improvement]
related_patterns: [pure-text-skill, cli-integration]
---

# 脚本/Hook 触发型 Skill

> 一句话定义：通过 shell 脚本在特定 agent 事件（用户发消息、工具调用完成、会话结束）时向 LLM 注入上下文，把"被动等调用"的 skill 变成"主动提醒"的持续行为。

## 本质

普通 skill 需要用户显式触发（"使用 self-improvement skill"）。Hook 型 skill 解决一个不同的问题：**让行为在对话过程中自动发生，不依赖用户记得触发**。

机制很简单：agent 框架（Claude Code / Codex）在特定事件发生时，执行一个 shell 脚本，将脚本的 stdout 注入到 LLM 的上下文中。LLM 读到这段注入内容，行为随之改变。

整个链路：`事件触发 → shell 脚本执行 → stdout 注入上下文 → LLM 行为改变`

## 什么时候用

- 需要在每次对话/任务结束后检查是否有可记录的学习（self-improvement 场景）
- 需要监控工具调用结果，在出错时自动提醒 LLM 记录错误（错误检测场景）
- 需要在会话开始时注入项目状态或规则提醒（session init 场景）
- 行为需要**持续**发生，而不是一次性触发

## 什么时候不用

- 只需要 LLM 一次性查阅某个工具的用法 → 用 CLI 型或纯文本型
- 行为是用户主动发起的（"帮我部署"）→ 普通 skill 就够了
- hook 脚本需要输出大量信息（超过 200 tokens）→ 会严重污染 context，重新设计

## 核心约束：脚本输出必须极简

**hook 脚本输出上限：50-100 tokens。**

这不是风格建议，是硬约束。原因：

UserPromptSubmit hook 在**每次用户发消息时**都执行。如果脚本输出 500 tokens，一个 20 轮对话就额外消耗了 10,000 tokens context，同时把真正重要的对话内容挤出窗口。

对比两种实现：

**错误做法（太冗长）：**
```
You are a self-improving AI assistant. After completing each task, you should 
carefully evaluate whether the interaction produced any learnings that could 
improve future performance. Consider the following categories:
1. Command failures and their root causes
2. User corrections that indicate gaps in your knowledge
... (继续 300 tokens)
```

**正确做法（self-improvement activator.sh 的实际输出）：**
```xml
<self-improvement-reminder>
After completing this task, evaluate if extractable knowledge emerged:
- Non-obvious solution discovered through investigation?
- Workaround for unexpected behavior?
- Project-specific pattern learned?
- Error required debugging to resolve?

If yes: Log to .learnings/ using the self-improvement skill format.
If high-value (recurring, broadly applicable): Consider skill extraction.
</self-improvement-reminder>
```

这个输出约 60 tokens。每轮 60 tokens 开销，20 轮对话 1,200 tokens——可接受。

**为什么用 XML 标签包裹输出？**

`<self-improvement-reminder>` 这个包裹标签有三个作用：
1. **语义隔离**：LLM 能区分"这是系统注入的提醒"和"这是用户说的话"
2. **条件识别**：LLM 可以学会忽略某类标签（如果用户明确表示不需要）
3. **调试可见**：开发者在 transcript 里能快速找到 hook 注入了什么

## 触发时机类型

Claude Code 支持四种 hook 事件：

| 事件 | 触发时机 | 典型用途 |
|------|---------|---------|
| `UserPromptSubmit` | 用户每次发消息之前 | 注入持续提醒、状态检查 |
| `PostToolUse` | 工具调用完成之后 | 检查工具输出、捕获错误 |
| `Stop` | LLM 完成一轮回复后 | 会话结束总结、清理工作 |
| `PreToolUse` | 工具调用开始之前 | 权限检查、输入验证 |

self-improvement skill 用了两个 hook：

```json
{
  "hooks": {
    "UserPromptSubmit": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "./skills/self-improvement/scripts/activator.sh"
      }]
    }],
    "PostToolUse": [{
      "matcher": "Bash",
      "hooks": [{
        "type": "command",
        "command": "./skills/self-improvement/scripts/error-detector.sh"
      }]
    }]
  }
}
```

- `activator.sh` 挂在 `UserPromptSubmit`：每次对话前提醒 LLM 评估是否有可记录的学习
- `error-detector.sh` 挂在 `PostToolUse (Bash)`：Bash 工具执行后检测输出是否包含错误关键词

**`matcher` 字段**：
- `""` = 匹配所有（UserPromptSubmit 没有可 match 的内容，留空即可）
- `"Bash"` = 只匹配 Bash 工具的调用（PostToolUse 可以按工具名过滤）

## 条件触发 vs 无条件触发

**无条件触发**（activator.sh）：每次都输出提醒。

适合：提醒开销极低（60 tokens），且每次任务后评估都有价值。

**条件触发**（error-detector.sh）：只在检测到错误时输出。

```bash
# error-detector.sh 的核心逻辑
OUTPUT="${CLAUDE_TOOL_OUTPUT:-}"

contains_error=false
for pattern in "${ERROR_PATTERNS[@]}"; do
    if [[ "$OUTPUT" == *"$pattern"* ]]; then
        contains_error=true
        break
    fi
done

if [ "$contains_error" = true ]; then
    cat << 'EOF'
<error-detected>
...
</error-detected>
EOF
fi
# 没有错误时：脚本输出为空，零 tokens 开销
```

`CLAUDE_TOOL_OUTPUT` 是 Claude Code 在 PostToolUse hook 执行时自动注入的环境变量，包含工具调用的完整输出。

**条件触发适合**：事件频繁（每次 Bash 调用都触发）但只需要在特定条件下提醒，避免噪音。

## 目录结构约定

```
skills/<skill-name>/
├── SKILL.md              # 主文档：skill 描述 + 核心逻辑 + hook 配置说明
├── scripts/              # hook 脚本目录
│   ├── activator.sh      # UserPromptSubmit hook
│   ├── error-detector.sh # PostToolUse hook
│   └── extract-skill.sh  # 辅助脚本（不是 hook，但属于 skill 功能）
├── hooks/                # 平台特定 hook 配置（可选）
│   └── openclaw/         # OpenClaw 平台的 hook 配置
├── assets/               # 模板文件、静态资源
└── references/           # 详细参考文档（hooks-setup.md 等）
```

**关键约定**：
- `scripts/` 放所有 shell 脚本，hook 脚本和辅助脚本都在这里
- `hooks/` 放特定平台的 hook 安装文件（如 OpenClaw），不是通用的
- 脚本命名要描述其触发时机或用途（`activator`、`error-detector`）

## Skill 不自动安装 Hooks

**这是最重要的设计约束**：skill 本身只提供脚本，不自动修改用户的 `.claude/settings.json`。

原因：
1. **安全边界**：自动修改 agent 配置是侵入性操作，必须用户知情并同意
2. **路径不可预测**：skill 安装位置因用户而异（`~/.claude/skills/` vs `./skills/`），脚本路径必须用户自己填写
3. **平台差异**：Claude Code 用 `.claude/settings.json`，Codex 用 `.codex/settings.json`，OpenClaw 有自己的 hook 机制

**正确做法**：SKILL.md 里提供配置模板，用户手动复制到自己的 settings.json，并填入正确的脚本路径。

```markdown
## Hook Integration

This is **opt-in**. To enable automatic reminders:

Create/update `.claude/settings.json`:

\`\`\`json
{
  "hooks": {
    "UserPromptSubmit": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "<your-skill-path>/scripts/activator.sh"
      }]
    }]
  }
}
\`\`\`

Replace `<your-skill-path>` with the actual path where you installed the skill
(e.g., `~/.claude/skills/self-improvement` or `./skills/self-improvement`).
```

## SKILL.md 模板

```markdown
---
name: <skill-name>
description: "<触发场景描述>. Use when: (1) <情况1>, (2) <情况2>."
---

# <Skill Name>

<一句话说明 skill 做什么>

## Core Behavior

<描述 skill 的核心操作：记录什么、输出什么、如何判断>

## Quick Reference

| Situation | Action |
|-----------|--------|
| <情况 1> | <动作 1> |
| <情况 2> | <动作 2> |

## Setup

### Manual Setup (No Hooks)

<不使用 hook 时的手动触发方式>

### Hook Integration (Opt-in)

Enable automatic <行为描述> through agent hooks.

**Claude Code / Codex:**

Create/update `.claude/settings.json`:

\`\`\`json
{
  "hooks": {
    "UserPromptSubmit": [{
      "matcher": "",
      "hooks": [{
        "type": "command",
        "command": "<skill-path>/scripts/activator.sh"
      }]
    }]
  }
}
\`\`\`

### Available Hook Scripts

| Script | Hook Type | Trigger Condition | Output Size |
|--------|-----------|------------------|-------------|
| `scripts/activator.sh` | UserPromptSubmit | Always | ~60 tokens |
| `scripts/error-detector.sh` | PostToolUse (Bash) | On error patterns | ~50 tokens |

## <Main Feature Section>

<skill 的主要内容：格式模板、操作步骤等>
```

## 脚本编写规范

### 基本结构

```bash
#!/bin/bash
# <Script Name> Hook
# Triggers on <EventType> to <目的描述>
# Keep output minimal (~50-100 tokens) to minimize overhead

set -e

# <条件检查逻辑（如果是条件触发）>

cat << 'EOF'
<xml-tag-name>
<极简提醒内容，不超过 100 tokens>
</xml-tag-name>
EOF
```

### 规范要点

1. **shebang + `set -e`**：脚本报错时立即退出，不产生乱码输出
2. **注释写明 hook 类型和目的**：便于维护
3. **提醒内容用问句或 bullet**：更容易被 LLM 快速扫描
4. **条件触发时空输出就是最好的输出**：脚本不输出 = 零 tokens 开销
5. **不要在 hook 脚本里做复杂操作**：hook 脚本只负责输出提醒，不负责执行任务

### 读取工具输出的变量

```bash
# PostToolUse hook 专用
OUTPUT="${CLAUDE_TOOL_OUTPUT:-}"    # 工具调用的完整输出

# PreToolUse hook 专用（Claude Code 实现可能不同，参考官方文档）
INPUT="${CLAUDE_TOOL_INPUT:-}"      # 工具调用的输入参数
```

## 与 Claude Code Hooks 的关系

Claude Code 的 hooks 系统是基础设施层，skill 的脚本是在这个基础上运行的内容层。

```
Claude Code Hooks System (基础设施)
  ├── settings.json 配置
  ├── 事件触发机制
  └── stdout 注入机制
        ↑
        │ 运行
        │
Skill 的 shell 脚本 (内容层)
  ├── activator.sh
  └── error-detector.sh
```

**Skill 设计者需要了解的 Claude Code hooks 机制**：
- Hook 脚本的 stdout 会被注入到 LLM 的下一轮 context
- 脚本的 stderr 不注入 context（可以用 stderr 做调试输出）
- 脚本超时或报错时，hook 静默失败（不影响正常对话）
- `settings.json` 可以放在项目级（`.claude/settings.json`）或全局（`~/.claude/settings.json`）

## 常见踩坑

### 1. 脚本输出太长，污染 context

**问题**：hook 脚本输出 300+ tokens，每轮对话都注入，20 轮后有效 context 被侵蚀 6000 tokens。

**修复**：把"提醒"和"参考内容"分开。hook 脚本只输出 50-100 tokens 的提醒，详细内容让 LLM 主动 Read SKILL.md 获取。

---

### 2. 无条件触发高频事件

**问题**：把详细提醒挂在 `PostToolUse (Bash)` 上无条件触发，每次 Bash 命令都注入 100 tokens。

**后果**：一个代码调试 session 可能有 30+ 次 Bash 调用，额外消耗 3000 tokens context。

**修复**：高频事件（PostToolUse）必须条件触发，只在满足特定条件（如检测到错误）时才输出。

---

### 3. Hook 脚本硬编码路径

**问题**：脚本内部用了绝对路径（`/Users/neo/.claude/skills/...`），其他用户安装后路径不匹配，脚本报错。

**修复**：脚本内部使用相对路径或 `$(dirname "$0")` 获取脚本所在目录。不依赖外部文件时完全没问题（self-improvement 的 activator.sh 直接输出，不读任何文件）。

---

### 4. SKILL.md 里没有说明"不自动安装 hook"

**问题**：用户以为安装 skill 就自动启用了 hook，实际上什么都没发生。

**修复**：在 SKILL.md 的 Hook Integration 节明确写"This is opt-in"，提供完整的 settings.json 配置模板，说明需要用户手动配置。

---

### 5. XML 标签名不够具体

**问题**：用 `<reminder>` 这种通用标签，多个 hook 输出混在一起，LLM 无法区分来源。

**修复**：用 skill 名作为 XML 标签的一部分：`<self-improvement-reminder>`、`<error-detected>`。LLM 能通过标签名理解注入来源，也便于调试时在 transcript 里查找。

## 来源

- `~/.claude/skills/self-improvement/SKILL.md` — hook 型 skill 的完整实现参考
- `~/.claude/skills/self-improvement/scripts/activator.sh` — UserPromptSubmit hook，无条件触发，~60 tokens 输出
- `~/.claude/skills/self-improvement/scripts/error-detector.sh` — PostToolUse hook，条件触发，读取 `CLAUDE_TOOL_OUTPUT`
- Claude Code 官方文档 hooks 章节 — `UserPromptSubmit` / `PostToolUse` / `Stop` / `PreToolUse` 事件规范
