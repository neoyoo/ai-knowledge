---
tool-category: skill-system
tags: [skill, capability, extension, plugin, discovery]
sources: [claude-code, deer-flow, openharness]
difficulty: beginner
related_wiki: [wiki/tool-system, wiki/agent-loop]
related_tools: [user-interaction, scheduling]
---

# Skill System Tools

> 让 agent 加载、执行和管理 skill——可复用的能力模块，实现功能的动态扩展。

## 工具列表

| 工具名 | 来源 | 用途 |
|--------|------|------|
| `Skill` | Claude Code | 执行一个命名 skill（slash command），只读，不能创建 |
| `skill` | OpenHarness | 读取 bundled/user/plugin skill 内容 |
| `skill_manage` | DeerFlow | 创建/编辑/删除 custom skills，带安全扫描 |
| `setup_agent` | DeerFlow | bootstrap 时创建 agent 配置，初始化 skill 集合 |

---

## Schema

### Claude Code — `Skill`

```json
{
  "name": "Skill",
  "description": "执行一个已注册的 skill（slash command）。skill 是只读的预定义能力模块，不支持在运行时创建或修改。",
  "input_schema": {
    "type": "object",
    "properties": {
      "name": {
        "type": "string",
        "description": "skill 名称，对应 slash command 名（如 'commit'、'review-pr'）。只接受顶层名，不支持路径语法。"
      },
      "args": {
        "type": "string",
        "description": "传递给 skill 的参数字符串，可选"
      }
    },
    "required": ["name"]
  }
}
```

**示例调用：**

```json
{
  "name": "commit",
  "args": "-m 'fix: resolve memory leak in session manager'"
}
```

```json
{
  "name": "review-pr",
  "args": "123"
}
```

**注意：** `name` 只接受顶层 skill 名，不支持 `path/to/skill` 这样的路径语法。子文档内容需要用 Read 工具直接读取 skill 文件。

---

### OpenHarness — `skill`

```json
{
  "name": "skill",
  "description": "读取 skill 内容。skill 可以来自 bundled（内置）、user（用户自定义）或 plugin（插件）三个来源。",
  "input_schema": {
    "type": "object",
    "properties": {
      "action": {
        "type": "string",
        "enum": ["read", "list", "execute"],
        "description": "操作类型：读取内容、列出可用 skill、或执行"
      },
      "skill_name": {
        "type": "string",
        "description": "skill 名称，action 为 read/execute 时必填"
      },
      "source": {
        "type": "string",
        "enum": ["bundled", "user", "plugin"],
        "description": "skill 来源，不指定时搜索所有来源"
      },
      "args": {
        "type": "object",
        "description": "执行参数，action 为 execute 时使用"
      }
    },
    "required": ["action"]
  }
}
```

**示例调用：**

```json
{
  "action": "list",
  "source": "user"
}
```

```json
{
  "action": "execute",
  "skill_name": "code-review",
  "source": "bundled",
  "args": { "file": "/project/src/main.py" }
}
```

---

### DeerFlow — `skill_manage`

```json
{
  "name": "skill_manage",
  "description": "在运行时创建、编辑或删除 custom skill。所有写操作都会经过安全扫描，拒绝恶意内容。",
  "input_schema": {
    "type": "object",
    "properties": {
      "action": {
        "type": "string",
        "enum": ["create", "edit", "delete", "list", "get"],
        "description": "操作类型"
      },
      "skill_name": {
        "type": "string",
        "description": "skill 名称，action 为 create/edit/delete/get 时必填"
      },
      "content": {
        "type": "string",
        "description": "skill 内容（Markdown 格式），action 为 create/edit 时必填"
      },
      "description": {
        "type": "string",
        "description": "skill 描述，action 为 create 时可选"
      },
      "tags": {
        "type": "array",
        "items": { "type": "string" },
        "description": "skill 标签，便于分类和搜索"
      }
    },
    "required": ["action"]
  }
}
```

**示例调用（创建新 skill）：**

```json
{
  "action": "create",
  "skill_name": "database-migration",
  "content": "# Database Migration\n\n## Steps\n1. Backup current schema\n2. Run migration scripts\n3. Verify integrity\n4. Update application config",
  "description": "执行数据库 schema 迁移的标准流程",
  "tags": ["database", "deployment", "migration"]
}
```

---

### DeerFlow — `setup_agent`

```json
{
  "name": "setup_agent",
  "description": "Bootstrap 时创建 agent 配置，初始化 agent 的 skill 集合和基本属性。",
  "input_schema": {
    "type": "object",
    "properties": {
      "agent_name": {
        "type": "string",
        "description": "agent 名称"
      },
      "description": {
        "type": "string",
        "description": "agent 功能描述"
      },
      "skills": {
        "type": "array",
        "items": { "type": "string" },
        "description": "为该 agent 启用的 skill 名称列表"
      },
      "system_prompt": {
        "type": "string",
        "description": "agent 的 system prompt 基础配置"
      },
      "tools": {
        "type": "array",
        "items": { "type": "string" },
        "description": "可用工具列表"
      }
    },
    "required": ["agent_name"]
  }
}
```

---

## 跨项目对比

| 维度 | Claude Code | OpenHarness | DeerFlow |
|------|-------------|-------------|----------|
| skill 执行 | 支持 | 支持 | 支持 |
| 运行时创建 skill | 不支持（只读） | 不支持（只读） | 支持（skill_manage） |
| 安全扫描 | N/A | N/A | 有（写操作必经） |
| skill 来源分层 | 单层（注册表） | bundled/user/plugin 三层 | 统一存储 |
| agent bootstrap | 无专用工具 | 无专用工具 | setup_agent |
| 参数传递 | 字符串 args | 对象 args | 对象 args |

**核心差异：静态 vs 动态。** Claude Code 和 OpenHarness 把 skill 当作"已部署的能力"——agent 只能选用，不能修改；DeerFlow 把 skill 当作"可演化的知识"——agent 可以在运行时创建和优化 skill，实现自我改进。这个设计差异有深刻影响：

- 静态方案：可预测、易审计、安全，适合生产系统
- 动态方案：灵活、可自适应，但需要安全扫描和版本控制配合

---

## 最佳实践

**skill 是导航员，不是百科全书。** 好的 skill 告诉 agent 做什么、用哪些工具、按什么顺序——而不是把所有知识塞进去。skill 应该短小精悍，指向正确的工具和参考资料。

**Claude Code skill 的 name 只用顶层名。** 常见错误：传入 `"name": "superpowers/commit"` 会失败。正确用法：`"name": "commit"`。如果需要读 skill 文件内容，直接用 Read 工具读取文件路径。

**DeerFlow 动态 skill 要做版本控制。** 运行时创建的 skill 如果不备份，agent 重启后可能丢失。重要 skill 应当导出到文件并纳入 git 管理。

**setup_agent 在 bootstrap 阶段完成，不要在运行时重复调用。** `setup_agent` 是一次性配置操作，在 agent 初始化时执行一次。运行时需要调整能力集，用 `skill_manage` 而不是重新 setup。

**先 list 再 execute。** 执行 skill 前先列出可用 skill，确认名称和参数格式，避免因拼写错误导致 silent fail。

---

## 关联

- [[cookbook/tools/definitions/user-interaction]] — skill 执行时可能需要通过 ask_user_question 收集参数
- [[cookbook/tools/definitions/scheduling]] — 定时任务常见用法是按计划触发执行某个 skill
- [[wiki/tool-system]] — skill 系统是 tool system 的高层抽象，将多工具流程封装为单一调用
- [[wiki/agent-loop]] — skill 调用发生在 agent 主循环的 action 阶段
