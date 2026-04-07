---
title: "Multi-Agent — DeerFlow"
category: L2
parent: "[[multi-agent]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 通过一个可选的 `task` 工具将主 agent 升级为 lead agent，启用后可派发子任务给独立的 SubagentExecutor 实例。子 agent 在线程池中异步并行运行，互相隔离，主 agent 通过 stream 事件轮询进度。架构上选择了"线程池 + async 事件循环"混合模型，在不修改核心 agent 逻辑的前提下实现真并行，并通过 middleware 硬性限制并发上限。

## 架构分析

### Lead Agent 与 task 工具

当 `subagent_enabled=True` 时，主 agent 的工具列表中注入一个 `task` 工具。调用签名为：

```python
task(description: str, prompt: str, subagent_type: str)
```

`subagent_type` 决定使用哪套子 agent 配置（内置类型：`general-purpose`、`bash`）。调用后在 `_background_tasks` 字典中注册任务，由 `get_stream_writer()` 持续向主流推送 `task_started` / `task_running` / `task_completed` 事件。

### SubagentExecutor 与双线程池

`SubagentExecutor` 维护两个 `ThreadPoolExecutor`：

- **scheduler pool**（3 workers）：调度与任务派发
- **execution pool**（3 workers）：实际执行，每个 worker 调用 `asyncio.run(_aexecute())`，内部是一次完整的 `create_agent` 调用，使用子 agent 自己的配置

子 agent 继承父 agent 的 `sandbox` 和 `thread_data`，但工具列表中不包含 `task` 工具，即不允许递归嵌套。

### 并发限制 middleware

`SubagentLimitMiddleware` 挂在主 agent 的 middleware 链上，拦截每次 model response，统计当前并发 task 调用数。单次 model response 中 task 调用超过 3 次时拒绝执行，防止失控的扇出。

### ACP 协议桥接外部 agent

`invoke_acp_agent` 工具支持将 ACP 兼容的外部 agent（如 Codex、Claude Code）作为子 agent 调用，协议层自动处理请求/响应格式转换。

### 进度可见性

主 agent 每隔 **5 秒**轮询 `_background_tasks`，通过 stream writer 将子任务状态实时透传给上层调用方，实现端到端可观测性。

### 关键代码路径

- `deerflow/tools/builtins/task_tool.py` — task 工具实现，背景任务注册与事件流
- `deerflow/subagents/executor.py` — SubagentExecutor，双线程池与 asyncio 桥
- `deerflow/subagents/builtins/general_purpose.py` — 内置 general-purpose 子 agent 配置
- `deerflow/agents/middlewares/subagent_limit_middleware.py` — 并发上限强制执行

## 设计亮点

- **线程池 + async 混合**：每个子 agent 在独立线程中运行完整的 async 事件循环，真并行且不干扰主 agent 的 async 上下文
- **ACP 协议桥接**：通过标准协议将外部 agent 系统（Codex、Claude Code）纳入子 agent 调度，无需改造外部系统
- **Stream 事件透传**：子 agent 进度实时可见，主 agent 可在等待期间继续其他工作，不阻塞
- **Middleware 限流**：并发上限不在业务逻辑里硬编码，而由独立 middleware 强制执行，关注点分离清晰

## 局限性

- **无递归嵌套**：子 agent 不能再派发子 agent，限制了深度分解复杂任务的能力
- **无 peer-to-peer 通信**：子 agent 之间不能直接交换数据，只能通过主 agent 中转
- **5 秒轮询延迟**：进度更新最多滞后 5 秒，对延迟敏感的场景不够精细
- **子 agent 无持久状态**：每次 task 调用创建全新的 agent 实例，无法跨调用积累上下文

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
