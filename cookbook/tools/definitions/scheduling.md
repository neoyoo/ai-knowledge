---
tool-category: scheduling
tags: [cron, schedule, timer, recurring, automation]
sources: [claude-code, openharness]
difficulty: intermediate
related_wiki: [wiki/agent-loop, wiki/tool-system]
related_tools: [user-interaction, skill-system]
---

# Scheduling Tools

> 让 agent 创建、管理和触发定时任务，实现无人值守的周期性自动化。

## 工具列表

| 工具名 | 来源 | 用途 |
|--------|------|------|
| `CronCreate` | Claude Code | 创建定时任务，支持 session-only 和 durable 两种持久化 |
| `CronList` | Claude Code | 列出当前所有定时任务 |
| `CronDelete` | Claude Code | 按 ID 删除定时任务 |
| `RemoteTrigger` | Claude Code | 立即触发远程 agent 或管理触发配置 |
| `cron_create` | OpenHarness | 创建定时任务，支持人类可读表达式 |
| `cron_list` | OpenHarness | 列出定时任务 |
| `cron_delete` | OpenHarness | 删除定时任务 |
| `remote_trigger` | OpenHarness | 立即触发远程 agent |

---

## Schema

### Claude Code — `CronCreate`

```json
{
  "name": "CronCreate",
  "description": "创建一个定时任务。可以是 session-only（会话结束后消失）或 durable（持久化到磁盘）。",
  "input_schema": {
    "type": "object",
    "properties": {
      "schedule": {
        "type": "string",
        "description": "cron 表达式，如 '*/5 * * * *'（每 5 分钟），或人类可读格式如 'every 5 minutes'"
      },
      "prompt": {
        "type": "string",
        "description": "定时触发时执行的 prompt 或命令"
      },
      "durable": {
        "type": "boolean",
        "description": "是否持久化（跨会话存活），默认 false（session-only）",
        "default": false
      },
      "name": {
        "type": "string",
        "description": "任务名称，便于后续识别"
      },
      "timezone": {
        "type": "string",
        "description": "时区，如 'Asia/Shanghai'，默认使用系统时区"
      }
    },
    "required": ["schedule", "prompt"]
  }
}
```

**示例调用：**

```json
{
  "schedule": "0 9 * * 1-5",
  "prompt": "检查 CI 状态并生成每日报告",
  "durable": true,
  "name": "daily-ci-report",
  "timezone": "Asia/Shanghai"
}
```

---

### Claude Code — `CronList`

```json
{
  "name": "CronList",
  "description": "列出所有活跃的定时任务，包括 session-only 和 durable。",
  "input_schema": {
    "type": "object",
    "properties": {
      "include_session_only": {
        "type": "boolean",
        "description": "是否包含 session-only 任务，默认 true",
        "default": true
      }
    }
  }
}
```

---

### Claude Code — `CronDelete`

```json
{
  "name": "CronDelete",
  "description": "按任务 ID 删除一个定时任务。",
  "input_schema": {
    "type": "object",
    "properties": {
      "cron_id": {
        "type": "string",
        "description": "要删除的任务 ID（来自 CronList 的结果）"
      }
    },
    "required": ["cron_id"]
  }
}
```

---

### Claude Code — `RemoteTrigger`

```json
{
  "name": "RemoteTrigger",
  "description": "立即触发一个远程 agent 执行，或管理触发器配置。",
  "input_schema": {
    "type": "object",
    "properties": {
      "action": {
        "type": "string",
        "enum": ["trigger", "create", "delete", "list"],
        "description": "操作类型"
      },
      "trigger_id": {
        "type": "string",
        "description": "触发器 ID，action 为 trigger/delete 时必填"
      },
      "prompt": {
        "type": "string",
        "description": "触发时执行的 prompt，action 为 create 时必填"
      },
      "name": {
        "type": "string",
        "description": "触发器名称，action 为 create 时可选"
      }
    },
    "required": ["action"]
  }
}
```

---

### OpenHarness — `cron_create`

```json
{
  "name": "cron_create",
  "description": "创建定时任务，支持标准 cron 表达式和人类可读的 schedule 格式。",
  "input_schema": {
    "type": "object",
    "properties": {
      "schedule": {
        "type": "string",
        "description": "调度表达式。支持 cron 格式（'0 * * * *'）或人类可读格式（'every hour', 'every day at 9am', 'every monday at 8:30'）"
      },
      "prompt": {
        "type": "string",
        "description": "触发时执行的内容"
      },
      "name": {
        "type": "string",
        "description": "任务名称"
      }
    },
    "required": ["schedule", "prompt"]
  }
}
```

**示例调用（人类可读格式）：**

```json
{
  "schedule": "every weekday at 9am",
  "prompt": "拉取最新代码，运行测试套件，汇报结果",
  "name": "morning-test-run"
}
```

---

### OpenHarness — `remote_trigger`

```json
{
  "name": "remote_trigger",
  "description": "立即触发远程 agent 执行指定任务。",
  "input_schema": {
    "type": "object",
    "properties": {
      "trigger_id": {
        "type": "string",
        "description": "触发器 ID"
      },
      "override_prompt": {
        "type": "string",
        "description": "可选，覆盖原始触发 prompt"
      }
    },
    "required": ["trigger_id"]
  }
}
```

---

## 跨项目对比

| 维度 | Claude Code | OpenHarness |
|------|-------------|-------------|
| cron 表达式 | 标准 5 字段格式 | 标准格式 + 人类可读格式 |
| 持久化 | session-only / durable 双模式 | 默认持久化 |
| 时区支持 | 支持（timezone 字段） | 跟随系统 |
| 立即触发 | RemoteTrigger | remote_trigger |
| 任务管理 | Create/List/Delete 三件套 | Create/List/Delete 三件套 |

**选型建议：**
- 用户不熟悉 cron 语法时优先 OpenHarness（人类可读表达式更友好）
- 需要区分会话生命周期任务和持久任务 → Claude Code 的 `durable` 标志
- 需要立即触发而不等到下次调度 → 两者均提供 remote_trigger

---

## 最佳实践

**durable vs session-only 的选择。** session-only 适合"这次调试期间每 30 秒检查一次进度"；durable 适合"每天早上 9 点执行代码审查"。错误使用 durable 会留下大量僵尸任务，定期用 CronList 清理。

**人类可读格式减少错误。** cron 表达式容易写错（`0 9 * * 1-5` 和 `9 0 * * 1-5` 完全不同）。OpenHarness 的 `every weekday at 9am` 格式对 agent 自动生成 schedule 时更安全。

**触发 prompt 要自包含。** 定时任务执行时没有上下文，prompt 必须包含完整指令，不能依赖"上文说过的"内容。

**用 name 标识任务。** 不命名的任务在 CronList 里只有 ID，几天后就不知道是干什么的了。

**创建前先 List。** 防止重复创建同名任务，尤其在重试场景下。

---

## 关联

- [[cookbook/tools/definitions/user-interaction]] — 定时任务完成后可用 SendUserMessage 通知用户
- [[cookbook/tools/definitions/skill-system]] — 定时任务常见用法是触发执行某个 skill
- [[wiki/agent-loop]] — 调度工具是 agent 主循环之外的异步触发机制
