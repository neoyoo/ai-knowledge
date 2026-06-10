---
title: "hooks——agentscope"
category: L2
parent: "[[hooks]]"
source: "agentscope"
source_version: "v2.0.1-11-g0e5418e8"
concept: "hooks"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的扩展点已经不是旧版 metaclass hook。当前主线是 **middleware lifecycle hooks**：`MiddlewareBase` 定义 reply、reasoning、acting、model call、context compression 和 system prompt 六个切入点，`Agent` 在运行时按 middleware 列表构造洋葱链或顺序 transformer pipeline。

当前源码中未找到旧页引用的 `agent/_agent_meta.py`、`AgentBase.register_class_hook()`、`_ReActAgentMeta` 等 metaclass hook 路径，因此旧的“实例级/类级 hook + OrderedDict + deepcopy”结论不再代表 2.x。

---

## 架构分析

### Hook 点

`MiddlewareBase` 当前暴露六类扩展点：

| Hook | 模式 | 作用 |
|---|---|---|
| `on_reply` | onion | 包裹完整 `Agent.reply_stream()` / `_reply_impl()` |
| `on_reasoning` | onion | 包裹模型推理阶段 |
| `on_acting` | onion | 包裹纯工具 I/O 执行层 |
| `on_model_call` | onion | 包裹原始模型 API 调用 |
| `on_compress_context` | onion | 包裹 `Agent.compress_context()` |
| `on_system_prompt` | transformer | 顺序改写 system prompt 字符串 |

`is_implemented(hook_name)` 会比较子类方法和基类占位方法是否相同，只把真正 override 的 hook 放入对应 middleware 列表。Agent 初始化时把 middleware 拆成 `_reply_middlewares`、`_reasoning_middlewares`、`_acting_middlewares`、`_model_call_middlewares`、`_compress_context_middlewares` 和 `_system_prompt_middlewares`。

### 两种执行模型

**Onion pattern** 用于可包裹的异步流程。以 reply 为例，`Agent._reply()` 递归构造 `next_handler()`，每层 middleware 可以在调用前后执行逻辑，也可以改变传给下一层的 `input_kwargs`。

```
middleware[0].on_reply(agent, input_kwargs, next_handler)
  -> middleware[1].on_reply(...)
      -> Agent._reply_impl(inputs=...)
```

**Transformer pattern** 用于 system prompt。多个 middleware 依序接收上一个 middleware 的输出字符串，最终结果再进入 model input。

### App 层内置 middleware

服务化运行时通过 `ChatService` 每轮装配 middleware：

- `InboxMiddleware`：从 message bus inbox 注入 team/background task 消息
- `StateChangeMiddleware`：把状态变化发布到 session event stream
- `ToolOffloadMiddleware`：对长工具任务做后台 offload / wakeup
- `extra_agent_middlewares`：应用方按 user/agent/session 动态追加

这说明 AgentScope 2.x 的 hook 主要服务 Web/distributed runtime，而不是单纯给 SDK 子类注入 pre/post 方法。

---

## 关键代码路径

- `src/agentscope/middleware/_base.py` — `MiddlewareBase` 六个 hook 定义和 `is_implemented()`
- `src/agentscope/agent/_agent.py` — `Agent.__init__()` 按 hook 类型拆分 middleware
- `src/agentscope/agent/_agent.py` — `_reply()` / `_reasoning()` / `_acting()` 构造 onion chain
- `src/agentscope/agent/_agent.py` — `compress_context()` 构造 `on_compress_context` chain
- `src/agentscope/app/_service/_chat.py` — App 层每轮注入 `InboxMiddleware` / `StateChangeMiddleware` / `ToolOffloadMiddleware`

---

## 设计亮点

### 1. Middleware 粒度对齐 agent loop

扩展点覆盖 reply、reasoning、model call、acting、context compression，基本对应 agent 主循环的关键阶段。相比只暴露 pre/post tool 或 pre/post reply，这种粒度更适合做观测、限流、策略注入和后台任务 offload。

### 2. `on_acting` 刻意只包裹纯 I/O

`on_acting` 的注释明确说明：权限检查、输入校验、context 写入都在 hook 外部完成；hook 只包裹 `toolkit.call_tool`。这降低了把工具执行 offload 到后台时破坏 agent context 的风险。

### 3. System prompt 用 transformer 而非 onion

system prompt 是字符串转换，不是事件流。AgentScope 把它设计成顺序 transformer pipeline，避免每个 middleware 都要处理 async generator 的复杂度。

### 4. App runtime 可以按 session 动态注入

`extra_agent_middlewares(user_id, agent_id, session_id)` 每轮运行时生成 middleware，适合多租户 Web agent：不同用户、不同 session 可以挂不同审计、策略或观测逻辑。

---

## 局限性

### 1. 无声明式注册 / 热重载机制

middleware 通过构造 `Agent(..., middlewares=[...])` 注入，不是用户配置文件级 hook。若要像 Claude Code/OpenHarness 那样由用户声明 shell/http hooks，需要在 App 壳层另建注册和热重载系统。

### 2. Hook 顺序由列表顺序隐式决定

框架没有 `priority`、`before/after`、`depends_on` 等声明式排序。多个 middleware 同时修改同一阶段输入时，顺序必须由装配方维护。

### 3. `on_acting` 看不到权限和 context 写入

这是安全上的刻意隔离，但也意味着如果想审计“权限判断 + 工具执行 + context 落盘”的完整闭环，需要组合 `on_acting`、agent event stream 和 permission/tool 层日志，单个 hook 不够。

### 4. Middleware 本身无失败策略元数据

hook 抛异常会沿主流程传播。框架没有 `fail_open/fail_closed`、timeout、retry 这类声明式策略，需要 middleware 自己实现。

---

## 对 agent-os 的借鉴

agent-os 如果同时支持本地代理和 Web 分布式代理，应把 hook 设计成两层：

- 本地 SDK 层：`MiddlewareBase` 风格的 Python/TS 原生 hook，覆盖 loop、model、tool、context
- 产品壳层：声明式 hook registry，处理用户配置、热重载、超时、失败策略和审计

核心抽象建议保留 `next_handler` onion 模式，但要额外加入 hook metadata：`priority`、`timeout_ms`、`failure_policy`、`scope(local/session/tenant)`。

---

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
- 核心文件：
  - `src/agentscope/middleware/_base.py`
  - `src/agentscope/agent/_agent.py`
  - `src/agentscope/app/_service/_chat.py`
  - `src/agentscope/app/middleware/_inbox_middleware.py`
  - `src/agentscope/app/middleware/_state_change_middleware.py`
  - `src/agentscope/app/middleware/_tool_offload_middleware.py`
