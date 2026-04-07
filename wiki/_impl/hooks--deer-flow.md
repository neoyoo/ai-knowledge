---
title: "Hooks — DeerFlow"
category: L2
parent: "[[hooks]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 没有传统意义上的 hooks 系统（无 shell/HTTP 事件钩子）。它用 **middleware pipeline** 作为扩展机制，每个 middleware 类暴露 `before_model`、`after_model`、`before_agent`、`after_agent`、`wrap_tool_call` 五个切入点，功能上覆盖了 hooks 的所有典型使用场景，且因为 middleware 可访问完整 agent 状态，表达能力远超传统 hooks。

## 架构分析

### Middleware 作为 hooks 替代

DeerFlow 的 middleware pipeline 在 agent 生命周期的关键节点插入回调：

| Middleware 方法 | 等价 hooks 语义 |
|---|---|
| `before_model` | pre_llm_call |
| `after_model` | post_llm_call |
| `before_agent` | agent_start |
| `after_agent` | agent_end |
| `wrap_tool_call` | pre/post_tool_use（含拦截能力）|

与 shell hooks 不同，middleware 方法接收完整的 agent context 对象，可以读写任意状态，而不仅是事件 payload。

### @Next/@Prev 定位机制

自定义 middleware 可通过装饰器声明插入位置：

```python
@Next(GuardrailMiddleware)   # 插到 GuardrailMiddleware 之后
@Prev(ClarificationMiddleware)  # 插到 ClarificationMiddleware 之前
class MyMiddleware(BaseMiddleware):
    ...
```

不需要修改 DeerFlow 内部代码，注册后框架自动按拓扑顺序组装 pipeline。

### GuardrailMiddleware — 工具调用拦截

`GuardrailMiddleware.wrap_tool_call` 是最接近 Claude Code `pre_tool_use` hook 的机制。调用任意工具前，向注册的 `GuardrailProvider` 查询策略：

- 返回 `allow` → 正常执行
- 返回 `deny(reason)` → 拦截并将原因写入 tool result
- `fail_open` 模式：provider 异常时放行；`fail_closed` 模式：provider 异常时拒绝

GuardrailProvider 是独立的策略引擎，可以是规则引擎、LLM 评估器或外部服务，与 middleware 解耦。

### ClarificationMiddleware — 人机交互拦截

当 agent 调用 `ask_clarification` 工具时，`ClarificationMiddleware` 拦截执行，挂起 agent 循环，等待人类输入后恢复。这是 DeerFlow 实现 human-in-the-loop 的核心路径，功能等价于 Claude Code 的 `PreToolUse` hook 触发 block 响应。

### 关键代码路径

- `deerflow/agents/middlewares/` — 所有 middleware 实现（含 GuardrailMiddleware、ClarificationMiddleware、SubagentLimitMiddleware）
- `deerflow/guardrails/` — GuardrailProvider 接口与内置实现
- `backend/docs/middleware-execution-flow.md` — Mermaid 时序图，完整 pipeline 执行流程

## 设计亮点

- **Middleware 比 hooks 更强大**：hooks 只传递事件 payload，middleware 可读写完整 agent 状态，支持更复杂的策略（如统计跨调用的累计 token 用量）
- **@Next/@Prev 无侵入插入**：第三方 middleware 可精确指定位置，不依赖全局配置文件或修改框架源码
- **Guardrail 作为独立策略引擎**：安全策略与 middleware 机制解耦，GuardrailProvider 可热替换，比"在每个 hook 里写安全逻辑"更整洁
- **类型安全**：Python 类而非 shell 脚本，IDE 可做静态检查，重构安全

## 局限性

- **无 shell-command hooks**：无法像 Claude Code 那样在工具调用前后触发外部命令，集成外部自动化系统必须写 Python middleware
- **无热重载**：新 middleware 需要重启服务才能生效，无法运行时动态挂载
- **无 LLM 评估策略**：GuardrailProvider 目前无内置 LLM 评估器，OpenHarness 的 prompt hooks（让 LLM 判断是否放行）需自行实现

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
