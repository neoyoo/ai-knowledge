---
title: "Hooks — OpenHarness"
category: L2
parent: "[[hooks]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness hooks 系统支持 4 种事件类型（`SESSION_START`、`SESSION_END`、`PRE_TOOL_USE`、`POST_TOOL_USE`）和 4 种 hook 类型（command、http、prompt、agent），通过 `settings.json` 或插件 `hooks.json` 声明。其中 `prompt` 和 `agent` 类型是独特设计——以自然语言描述的 LLM 评估条件作为 hook 逻辑，配合 `HookReloader` 的 mtime 轮询实现热重载，无需重启即可更新 hook 配置。

## 架构分析

### 事件模型

4 个事件类型覆盖会话生命周期的关键节点：`SESSION_START`（会话初始化）、`SESSION_END`（会话结束）、`PRE_TOOL_USE`（工具执行前）、`POST_TOOL_USE`（工具执行后）。工具级 hook 支持可选的 `matcher` 字段，使用 fnmatch glob 语法匹配 `tool_name`，实现细粒度的工具过滤。

### Hook 类型

| 类型 | 执行方式 | 特点 |
|------|---------|------|
| `command` | 以 `/bin/bash -lc` 执行 shell 命令，通过 `OPENHARNESS_HOOK_PAYLOAD` 环境变量传入 JSON payload | 标准副作用执行 |
| `http` | POST JSON payload 到指定 URL | 外部系统集成 |
| `prompt` | 调用模型，结构化验证返回 `{"ok": true}` | LLM 条件判断 |
| `agent` | 类似 prompt，模型评估后决策 | LLM 条件判断（增强版） |

`block_on_failure: true` 标志可让 `PRE_TOOL_USE` hook 失败时阻断工具执行，实现 policy enforcement。

### LLM 评估型 Hook（prompt / agent）

`PromptHookDefinition` 允许用自然语言描述 hook 条件，例如 `"Block if file contains hardcoded credentials"`，系统将工具调用上下文传给模型，期望返回 `{"ok": true/false}` 结构化响应。`AgentHookDefinition` 在此基础上增加了更完整的 agent 上下文。这使得无法用 glob/regex 表达的语义条件可以直接编码为 hook。

### 热重载机制

`HookReloader` 在每轮对话开始时检查 `hooks.json` 和相关配置文件的 mtime，发现变更时重新加载 hook 定义，无需重启进程。这使 hook 调试和迭代周期极短。

### 关键代码路径

- `hooks/events.py` — 事件类型枚举与 payload schema
- `hooks/schemas.py` — 四种 hook 类型的 Pydantic schema（含 `PromptHookDefinition`、`AgentHookDefinition`）
- `hooks/loader.py` — 从 `settings.json` 和插件 `hooks.json` 加载并合并 hook 定义
- `hooks/executor.py` — hook 执行器，按类型分发到 command/http/prompt/agent 执行路径
- `hooks/hot_reload.py` — `HookReloader`：mtime 轮询 + 按需重载

## 设计亮点

- **LLM 评估型 hook**：`prompt` 和 `agent` 类型允许用自然语言描述策略条件，无需编写 shell/Python 脚本即可实现语义级 policy enforcement，是与 Claude Code 纯 command hook 的核心差异
- **热重载**：mtime 轮询方案简单有效，开发和调试 hook 时无需重启 agent 进程
- **插件集成**：hook 可由插件 `hooks.json` 声明，与 plugin 生态统一，实现 hook 的模块化分发

## 局限性

- **事件类型稀少**：仅 4 种事件，相比 Claude Code 的 28+ 事件（涵盖模型采样前后、工具分类等细粒度节点），可观测和可干预的点非常有限
- **无 schema 验证层**：缺乏类似 Zod 的运行时 schema 校验，hook payload 的结构正确性依赖调用方自律
- **fnmatch vs 完整模式匹配**：`matcher` 仅支持 fnmatch glob，无法表达复杂的工具名匹配规则（如正则、多条件组合）
- **串行执行**：所有 hook 顺序执行，无并发调度，延迟敏感场景（如多个 http hook）性能受限

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
