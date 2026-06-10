---
name: neoagent llm context projection demo
description: 面向 neoagent 的 LLM 可见上下文 demo。采用 Markdown-first：上下文主体保持可读 section，SDK 内部结构和观测 ID 不直接泄露到 prompt。
type: design-demo
status: inbox
date: 2026-05-02
relates_to:
  - ideas/2026-05-02-neoagent-context-protocol-v2.md
  - wiki/_patterns/tool-metadata-driven-context-lifecycle.md
  - wiki/context-management.md
  - wiki/prompt-system.md
  - wiki/memory-system.md
---

# neoagent LLM 上下文投影 Demo

> 目标：先不考虑 SDK 怎么实现，只看每轮真正传给 LLM 的上下文应该长什么样。
>
> 核心原则：LLM 可见上下文优先服务推理和协作，可读性高于内部结构完整性。SDK 可以在内部维护 `ContextBlock`、`ContextEvent`、trace、compression、schema registry，但不要把这些对象原样 dump 到 prompt。

---

## 0. 这版 Demo 的纠偏

上一版把太多 SDK 内部协议字段放进了 LLM 可见上下文，例如 `trace_id`、`span_id`、`event_id`、`compression_id`、`priority`、`max_tokens`、`lifecycle`、`schema_id`。这会让上下文读起来像数据库记录，反而不如原来的固定 8 层清晰。

新的方向是：

- LLM 看到的是 **Markdown section**，不是内部对象 dump。
- XML-like tags 只用于少量 provider / skill / command metadata，不作为上下文主体语言。
- 每个上下文 section 最多带一个轻量 handle，供 LLM 更新、引用或 recall。
- trace、span、event、compression、budget、schema registry 都留在 runtime 内部。
- 原来的 8 层不作为核心架构固定下来，但可以作为默认可读 layout 的灵感。

一句话：

> 内部结构化，外部可读化。

---

## 1. Provider 请求形态

neoagent 每轮给 provider 的请求可以抽象成：

```text
LLMRequest {
  system:   runtime contract + capability summary + context projection
  messages: active messages + skill meta messages + reminders
  tools:    runtime tools + Skill tool + context protocol tools
}
```

其中：

- `system` 放稳定规则、权限边界、能力摘要和派生上下文投影。
- `messages` 放最近仍活跃的原始对话，是最近事实源。
- `tools` 放模型可调用动作，包括 Skill 工具和上下文协议工具。
- `skills`、`tools`、`MCP servers` 属于 capability plane，不属于 context projection。
- `context projection` 只放派生状态，例如当前任务摘要、决策、压缩历史、项目记忆、可召回产物。

---

## 2. LLM 可见 ID 的最小规则

LLM 可见上下文里的 ID 只保留一类东西：

> 模型需要拿来引用、更新、加载或恢复的 handle。

### 2.1 默认可见的 handle

| Handle | 用途 | 示例 |
|--------|------|------|
| `ctx:*` | 指向一个可更新的上下文 section | `ctx:task.goal` |
| `recall:*` | 指向可恢复的压缩历史、工具产物或长内容 | `recall:history.001` |
| skill name | 供模型调用 Skill 工具加载能力指令 | `context_protocol_design` |
| tool name | provider-native tool 名称 | `recall_context` |

### 2.2 默认不进入 LLM prompt 的 ID

这些 ID 由 runtime / SDK / observability 系统维护，默认不渲染给 LLM：

| 内部 ID | 为什么不默认展示 |
|---------|------------------|
| `session_id` / `task_id` / `turn_id` | runtime envelope 已经知道，模型调用工具时不需要手写 |
| `trace_id` / `span_id` / `traceparent` | OTel / Langfuse 链路追踪字段，runtime 自动传播 |
| `event_id` | event log 内部审计字段 |
| `compression_id` | compressor 内部对象 ID，LLM 只需要 `recall:*` |
| `schema_id` | schema registry 内部版本 ID，普通 prompt 不需要 |
| `projection_id` | renderer 内部投影 ID |
| `artifact_id` | artifact store 内部 ID，LLM 只需要 `recall:*` |
| `skill_invocation_id` | capability runtime 内部调用记录 |

### 2.3 message_id 的处理

`message_id` 是 canonical message 的稳定 ID，但不应该默认塞进每个上下文 section。

推荐策略：

- 普通上下文：不展示 `message_id`，只展示可读摘要。
- 需要引用来源时：用自然语言写来源，例如“来自上一轮用户澄清”。
- debug / audit 模式：可以渲染 `source: msg_0005_user`。
- recall 工具内部：runtime 可以用 `message_id` 定位原始消息，不要求模型手写。

---

## 3. 推荐的 LLM 可见上下文形态

核心形态是 Markdown section：

```md
# Runtime Contract

## Identity
...

## Safety
...

# Capability Plane

## Tools
...

## Skills
...

# Active Task Context

## Task Goal
<!-- ctx:task.goal -->
...

## Decisions
<!-- ctx:task.decisions -->
...

## Open Questions
<!-- ctx:task.open_questions -->
...

# Compressed History

## Earlier Discussion
<!-- recall:history.001 -->
...
```

这里的 `<!-- ctx:... -->` 和 `<!-- recall:... -->` 是轻量 handle：

- 人类读文档时几乎不受干扰。
- LLM 可以看到并用于 tool call。
- SDK 可以把它映射回内部 `ContextBlock`、`ContextEvent`、message refs 和 observability span。

如果某个 provider 对 HTML comment 处理不稳定，可以切换成显式但仍然轻量的写法：

```md
## Decisions
Handle: `ctx:task.decisions`
```

---

## 4. 上下文默认 Layout

默认 layout 可以保留原来 8 层的可读性，但不要把“8 层”写死成 SDK 核心模型。

推荐默认顺序：

1. Runtime Contract
2. Capability Plane
3. Active Message Contract
4. Active Task Context
5. Compressed History
6. Project / Cross-session Memory
7. Recallable Artifacts
8. Update Rules

这 8 个是 **默认渲染模板**，不是不可变协议层。

任务简单时可以只渲染前 3 个部分。任务复杂时，`Active Task Context` 内部可以由模型声明更细的 working sections，例如：

- `ctx:task.goal`
- `ctx:task.constraints`
- `ctx:task.decisions`
- `ctx:task.failed_attempts`
- `ctx:task.open_questions`
- `ctx:task.next_steps`

---

## 5. 上下文元协议

框架固定这些边界：

- identity
- security
- tool registry
- skill registry and invocation policy
- MCP connections
- active messages
- event log
- persistence
- recall
- compression
- observability

模型可以影响这些内容：

- 当前任务需要哪些 working sections。
- 每个 section 的标题和用途。
- 某个 section 是替换、追加，还是追加后修剪。
- 是否需要召回压缩历史或大型产物。
- 是否需要请求 memory context。
- 是否需要发现或加载 skill。

但这些影响应通过 tool call 表达，不应该靠模型手写内部 metadata。

### 5.1 声明任务上下文 section

```json
{
  "name": "declare_context_sections",
  "arguments": {
    "sections": [
      {
        "handle": "ctx:task.goal",
        "title": "任务目标",
        "purpose": "记录当前任务的目标和判断标准",
        "update": "replace"
      },
      {
        "handle": "ctx:task.decisions",
        "title": "已确认决策",
        "purpose": "记录后续必须遵守的设计决策",
        "update": "append"
      },
      {
        "handle": "ctx:task.open_questions",
        "title": "未解决问题",
        "purpose": "记录尚未确认、需要继续和用户讨论的问题",
        "update": "append_prune"
      }
    ]
  }
}
```

注意：

- 模型声明的是 LLM 可见 section，不是内部 schema 对象。
- runtime 可以在内部生成 `schema_id`、`event_id` 和版本记录。
- prompt 里不需要展示 `schema_id`。

### 5.2 更新上下文 section

```json
{
  "name": "update_context_section",
  "arguments": {
    "handle": "ctx:task.decisions",
    "operation": "append",
    "content": "- LLM 可见上下文使用 Markdown-first；内部 ContextBlock 不直接渲染成 prompt。"
  }
}
```

runtime 接收后负责：

- 校验 `handle` 是否存在。
- 记录 event log。
- 关联当前 session / task / turn / trace。
- 更新内部 context state。
- 下轮渲染成可读 Markdown。

---

## 6. LLM 可见请求示例 A：新任务开始

这是第一次调用时，LLM 可能看到的 `system + meta messages` 组合。示例中为了便于阅读放在一个 Markdown 块里；实际 provider request 可以拆成 system、developer、user meta messages。

```md
# Runtime Contract

## Identity

你是 neoagent，一个 AI 工程智能体。你负责帮助用户设计、修改和评估 agent 系统。

## Safety

- 遵守用户和系统指令。
- 不暴露密钥、凭证或其他敏感信息。
- 除非用户明确要求实现或修改，否则不要改动文件。
- 破坏性操作必须得到用户明确批准。

## Authority

当信息冲突时，按这个顺序处理：

1. Runtime contract and safety
2. Capability contracts: tools, skills, MCP
3. 当前用户消息
4. Active messages
5. Invoked skill content
6. Compressed history
7. Active task context
8. Memory context

# Capability Plane

## Tools

- read repository files
- draft Markdown documents
- declare_context_sections
- update_context_section
- request_memory_context
- recall_context

## Skills

可通过 Skill 工具加载：

- context_protocol_design: 设计 LLM 上下文协议和上下文投影
- kb_review: 审阅知识库文档的一致性、溯源和结构质量

# Active Message Contract

最近的用户消息和 assistant 回复由 provider messages 提供，是最近事实源。
如果下面的派生上下文与 active messages 冲突，优先相信 active messages。

# Project Memory

## neoagent context protocol notes
<!-- recall:memory.neoagent_context_protocol_v2 -->

- 之前已有 context-protocol v2 草案。
- 已有方向：messages 是 canonical；working state 是 derived。
- 当前讨论希望从固定 working-memory 字段转向更灵活的上下文投影。
```

### 模型可能采取的动作

新任务如果需要跨轮次维护上下文，模型可以声明本任务的 working sections：

```json
{
  "name": "declare_context_sections",
  "arguments": {
    "sections": [
      {
        "handle": "ctx:task.goal",
        "title": "任务目标",
        "purpose": "记录当前要完成的设计目标",
        "update": "replace"
      },
      {
        "handle": "ctx:task.decisions",
        "title": "已确认决策",
        "purpose": "记录用户和模型已经确认的协议方向",
        "update": "append"
      },
      {
        "handle": "ctx:task.open_questions",
        "title": "未解决问题",
        "purpose": "记录需要继续讨论或确认的问题",
        "update": "append_prune"
      }
    ]
  }
}
```

---

## 7. LLM 可见请求示例 B：长任务中段

这是第 6 轮左右传给 LLM 的上下文。此时已经有 task context、压缩历史、skill 内容和可召回 artifact。注意：上下文主体仍然是 Markdown section，不是满屏 metadata。

````md
# Runtime Contract

## Identity

你是 neoagent，一个 AI 工程智能体。你负责帮助用户设计、修改和评估 agent 系统。

## Safety

- 遵守用户和系统指令。
- 不暴露密钥、凭证或其他敏感信息。
- 未经用户明确要求，不修改文件。

## Authority

runtime/security > capability contracts > current user message > active messages > invoked skills > compressed history > task context > memory context

# Capability Plane

## Tools

- declare_context_sections
- update_context_section
- request_memory_context
- recall_context
- discover_skills
- Skill

## Skills

可通过 Skill 工具加载：

- context_protocol_design
- kb_review

# Invoked Skill

<command-message>context_protocol_design</command-message>
<command-name>context_protocol_design</command-name>
<skill-format>true</skill-format>

Base directory for this skill: /skills/context-protocol

## Context Protocol Design

- 先定义模型可见的 context projection，再反推 SDK。
- Markdown 负责规则、角色、说明和正文。
- XML-like tags 只负责少量机器边界和 metadata。
- 不要把内部 ContextBlock / trace / event log 原样暴露给 LLM。

# Active Task Context

## 任务目标
<!-- ctx:task.goal -->

设计 neoagent 的 LLM 可见上下文协议。当前阶段只关心每轮传给 LLM 的上下文长什么样，先不进入 SDK 实现。

判断标准：

- 人和 LLM 都容易读。
- 保留足够少的 handle，支持 tool call 更新和 recall。
- tools、skills、MCP 属于 capability plane。
- runtime 可以做观测、压缩和恢复，但不把内部 ID 噪声暴露给 LLM。

## 已确认决策
<!-- ctx:task.decisions -->

- LLM 可见上下文采用 Markdown-first。
- XML-like tags 只用于少量 skill / command / provider metadata，不作为上下文主体。
- 原来的 8 层可以作为默认 layout 灵感，但不作为 SDK 核心模型硬编码。
- `ctx:*` 是可更新 section handle。
- `recall:*` 是可恢复内容 handle。
- `trace_id`、`span_id`、`event_id`、`compression_id`、`schema_id` 默认不进入 prompt。
- Skill 与 tools、MCP 同属 capability plane，不放入 context projection。

## 未解决问题
<!-- ctx:task.open_questions -->

- `ctx:*` handle 使用 HTML comment，还是显式 `Handle: ...` 行？
- 默认 layout 是否保留 8 个一级 section，还是按任务类型缩减？
- memory context 默认放在 system，还是作为 meta message 注入？

## 下一步
<!-- ctx:task.next_steps -->

把当前 demo 文档改成 Markdown-first 的 LLM 可见上下文样例，删除旧版 metadata-heavy 的 `context-block` 示例。

# Compressed History

## 早期讨论摘要
<!-- recall:history.001 -->

前面的讨论从固定 8 层上下文转向 meta-protocol。随后发现如果把 `ContextBlock`、`schema_id`、trace、compression、budget 等内部字段直接暴露给 LLM，可读性会显著下降。

当前修正方向是：内部仍然结构化，外部渲染成类似原 8 层的 Markdown section。模型只看到少量 `ctx:*` 和 `recall:*` handle。

如需恢复早期原文，调用：

```text
recall_context(ref="recall:history.001", reason="需要核对早期决策")
```

# Project Memory

## neoagent context protocol v2 notes
<!-- recall:memory.neoagent_context_protocol_v2 -->

- v2 草案已经区分 canonical messages 和 derived views。
- v2 的问题是 working-memory 字段偏固定。
- 新方向不是完全抛弃可读层次，而是把固定层次降级成默认 renderer layout。

# Recallable Artifacts

| Ref | Preview |
|-----|---------|
| `recall:artifact.context_protocol_v2` | `ideas/2026-05-02-neoagent-context-protocol-v2.md` |
| `recall:artifact.claude_skill_loading` | Claude Code skill loading 相关源码摘要 |

只有当 preview 不足以支持当前判断时，才调用 `recall_context(ref=...)`。

# Update Rules

- 需要修改 task context 时，调用 `update_context_section(handle=...)`。
- 如果 task context 与 active messages 冲突，优先相信 active messages，并更新 task context。
- 不要把 memory context 当成事实源；它只是 advisory background。
- 不要生成、修改或传播 trace/span/traceparent；这些由 runtime 处理。
````

---

## 8. Skill 的上下文形态

Skill 不属于 context projection，而属于 capability plane。

Claude Code 的模式可以抽象为：

1. skill listing：低成本 Markdown 列表，告诉模型有哪些 skill。
2. Skill tool call：模型按名称请求加载完整 skill。
3. invoked skill content：完整 skill Markdown 作为 meta user message 注入，前面带少量 XML-like command metadata。

neoagent 可以采用同样思路：

```md
## Skills

可通过 Skill 工具加载：

- context_protocol_design
- kb_review
```

调用后：

```md
<command-message>context_protocol_design</command-message>
<command-name>context_protocol_design</command-name>
<skill-format>true</skill-format>

Base directory for this skill: /skills/context-protocol

# Context Protocol Design

...
```

不要把 skill 渲染成：

```md
<context-block kind="skill" ...>
...
</context-block>
```

原因是 skill 的身份、版本、权限、调用策略由 capability runtime 管理，不是派生上下文。

---

## 9. SDK 内部对象与 LLM 可见上下文的关系

LLM 可见上下文是 Markdown，但 SDK 内部仍然应该结构化。

推荐内部对象：

```text
ContextProjection
ContextSection
ContextEvent
RecallEntry
CapabilityRegistry
SkillInvocation
ObservabilityEnvelope
```

关键边界：

| 内部需要 | LLM 是否默认看到 |
|----------|------------------|
| session / task / turn 归属 | 否 |
| trace / span / traceparent | 否 |
| event log | 否 |
| schema registry | 否 |
| compression object | 否 |
| context section handle | 是，`ctx:*` |
| recall handle | 是，`recall:*` |
| skill name | 是 |
| tool name | 是 |

也就是说，SDK 可以在内部把这一段：

```md
## 已确认决策
<!-- ctx:task.decisions -->

- LLM 可见上下文采用 Markdown-first。
```

映射成结构化对象：

```text
ContextSection {
  handle: "ctx:task.decisions",
  title: "已确认决策",
  kind: "task_context",
  content: "...",
  internal: {
    session_id,
    task_id,
    turn_id,
    event_ids,
    source_message_ids,
    trace_context,
    update_policy
  }
}
```

但渲染给 LLM 时，只显示标题、正文和 handle。

---

## 10. 上下文协议工具

v0.1 工具可以保持少量：

```json
[
  {
    "name": "declare_context_sections",
    "description": "为当前任务声明需要长期维护的 Markdown sections。runtime 会校验 handle、标题和更新方式。"
  },
  {
    "name": "update_context_section",
    "description": "更新一个已声明的 context section。模型只传 handle、operation 和 content。"
  },
  {
    "name": "request_memory_context",
    "description": "按 intent 请求项目或跨会话记忆。返回内容是 advisory，不是 canonical。"
  },
  {
    "name": "recall_context",
    "description": "用 recall:* handle 恢复压缩历史、大型 artifact 或长内容。"
  },
  {
    "name": "discover_skills",
    "description": "按 intent 查询可用 skills，返回低成本 skill listing。"
  },
  {
    "name": "Skill",
    "description": "按 skill name 加载 framework-registered skill。"
  }
]
```

不建议 v0.1 暴露这些工具或参数：

- `set_trace_context`
- `set_budget`
- `set_priority`
- `set_schema_id`
- `set_compression_id`
- 要求模型传 `session_id` / `task_id` / `turn_id`

这些都应该由 runtime envelope 自动处理。

---

## 11. 设计结论

neoagent context protocol 不应该说“有固定 8 层”，也不应该把内部 `ContextBlock` metadata 全部展示给 LLM。

更合理的表达是：

> 一次 model call 接收 runtime contract、capability plane、active messages 和 Markdown-first context projection。Context projection 由可读 section 组成，section 可以带少量 `ctx:*` 或 `recall:*` handle。SDK 内部负责结构化、观测、压缩、恢复和持久化，但这些内部 ID 默认不进入 LLM prompt。

这样同时保留两件事：

- 原 8 层的可读性。
- meta-protocol 的灵活性。

---

## 12. 未决问题

- `ctx:*` handle 用 HTML comment 还是显式 `Handle: ...` 行？
- 默认 layout 是否固定为 8 个一级 section，还是按任务类型动态裁剪？
- `message_id` 是否只在 debug / audit 模式渲染？
- memory context 是放 system，还是作为 meta user message 注入？
- `declare_context_sections` 是否需要允许模型重命名 section title？
