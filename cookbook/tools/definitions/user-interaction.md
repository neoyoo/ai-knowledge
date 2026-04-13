---
tool-category: user-interaction
tags: [ask-user, question, clarification, communication]
sources: [claude-code, deer-flow, openharness]
difficulty: beginner
related_wiki: [wiki/query-loop, wiki/channel-interface]
related_tools: [scheduling, skill-system]
---

# User Interaction Tools

> 让 agent 在执行过程中主动向用户提问、请求确认或发送消息的工具集。

## 工具列表

| 工具名 | 来源 | 用途 |
|--------|------|------|
| `AskUserQuestion` | Claude Code | 结构化多选题，支持多选和 file preview |
| `SendUserMessage` | Claude Code | 向用户发送通知消息 |
| `ask_clarification` | DeerFlow | 分类型澄清问题，带 options |
| `ask_user_question` | OpenHarness | 简单问答 |

---

## Schema

### Claude Code — `AskUserQuestion`

```json
{
  "name": "AskUserQuestion",
  "description": "向用户提出一个或多个结构化问题，支持单选、多选，以及文件预览。",
  "input_schema": {
    "type": "object",
    "properties": {
      "questions": {
        "type": "array",
        "description": "问题列表，1-4 个",
        "items": {
          "type": "object",
          "properties": {
            "question": {
              "type": "string",
              "description": "问题文本"
            },
            "options": {
              "type": "array",
              "description": "可选项列表（如有）",
              "items": { "type": "string" }
            },
            "allow_multiple": {
              "type": "boolean",
              "description": "是否允许多选，默认 false"
            },
            "preview_file": {
              "type": "string",
              "description": "可选，展示给用户预览的文件路径"
            }
          },
          "required": ["question"]
        },
        "minItems": 1,
        "maxItems": 4
      }
    },
    "required": ["questions"]
  }
}
```

**示例调用：**

```json
{
  "questions": [
    {
      "question": "你希望使用哪种部署方式？",
      "options": ["Docker", "bare metal", "Kubernetes"],
      "allow_multiple": false
    },
    {
      "question": "需要启用哪些可选功能？",
      "options": ["监控", "日志聚合", "自动扩缩容"],
      "allow_multiple": true
    }
  ]
}
```

---

### Claude Code — `SendUserMessage`

```json
{
  "name": "SendUserMessage",
  "description": "向用户发送一条通知或摘要消息，不等待回复。",
  "input_schema": {
    "type": "object",
    "properties": {
      "message": {
        "type": "string",
        "description": "消息内容，支持 markdown"
      }
    },
    "required": ["message"]
  }
}
```

---

### DeerFlow — `ask_clarification`

```json
{
  "name": "ask_clarification",
  "description": "向用户请求澄清，带类型分类和结构化选项。",
  "input_schema": {
    "type": "object",
    "properties": {
      "question": {
        "type": "string",
        "description": "澄清问题的具体内容"
      },
      "clarification_type": {
        "type": "string",
        "enum": [
          "missing_info",
          "ambiguous",
          "approach_choice",
          "risk_confirmation",
          "suggestion"
        ],
        "description": "问题类型：缺少信息/含义模糊/方案选择/风险确认/建议"
      },
      "options": {
        "type": "array",
        "description": "可选项，当类型为 approach_choice 时尤为重要",
        "items": { "type": "string" }
      },
      "context": {
        "type": "string",
        "description": "背景说明，帮助用户理解为什么要问这个问题"
      }
    },
    "required": ["question", "clarification_type"]
  }
}
```

**示例调用：**

```json
{
  "question": "数据库迁移失败时，是否继续执行后续步骤？",
  "clarification_type": "risk_confirmation",
  "options": ["继续（忽略错误）", "停止并回滚", "停止等待人工处理"],
  "context": "检测到迁移脚本包含不可逆的 DROP TABLE 操作"
}
```

---

### OpenHarness — `ask_user_question`

```json
{
  "name": "ask_user_question",
  "description": "向用户提出一个简单问题并等待回答。",
  "input_schema": {
    "type": "object",
    "properties": {
      "question": {
        "type": "string",
        "description": "问题内容"
      }
    },
    "required": ["question"]
  }
}
```

---

## 跨项目对比

| 维度 | Claude Code | DeerFlow | OpenHarness |
|------|-------------|----------|-------------|
| 多题合并 | 支持（1-4 题） | 每次一题 | 每次一题 |
| 选项支持 | 有，支持多选 | 有，通过 options 字段 | 无 |
| 问题分类 | 无 | 有（5 种 type） | 无 |
| 文件预览 | 支持 preview_file | 无 | 无 |
| 风险提示 | 无内置 | risk_confirmation 类型 | 无 |
| 复杂度 | 中 | 中 | 低 |

**选型建议：**
- 需要多选、批量问题 → Claude Code `AskUserQuestion`
- 需要明确标注风险或决策类型 → DeerFlow `ask_clarification`
- 简单 yes/no 或文本输入 → 三者均可，OpenHarness 最简单

---

## 最佳实践

**只在真正需要时打断用户。** agent 在能够自主决策时应避免提问；只有在缺少关键信息、存在不可逆风险、或有多个等价方案需要人决策时才发起提问。

**提供选项比开放问题好。** 用户面对选项列表比面对空白输入框更容易回答，也能减少解析歧义。如果可能，把问题设计成 "A 还是 B？" 而不是 "你想怎么做？"

**问题要具体，说清楚背景。** 差的问题："你确认吗？" 好的问题："检测到目标目录非空（共 234 个文件），是否继续写入？将覆盖同名文件。"

**合并相关问题。** Claude Code 支持一次发送 1-4 个问题，利用这一点减少打断次数。把一组相关决策合并成一次交互。

**区分阻塞和通知。** `SendUserMessage` 不等回复，适合进度报告；`AskUserQuestion` 会阻塞 agent，只用在真正需要答案才能继续的地方。

---

## 关联

- [[cookbook/tools/definitions/skill-system]] — skill 执行时可能需要用 ask_user_question 收集参数
- [[wiki/channel-interface]] — 用户交互工具的底层通道抽象
- [[wiki/query-loop]] — agent 主循环中人机交互发生的位置
