---
title: "Query Loop — DeerFlow"
category: L2
parent: "[[query-loop]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 不自己实现 ReACT 循环，而是将循环本体委托给 LangGraph 的 `create_agent()`，自身的创新集中在**中间件管道**上。14 个中间件以 `before_agent` / `after_model` / `wrap_tool_call` 三类钩子介入循环的每个关键节点，在不修改核心循环代码的前提下注入横切关注点（loop detection、clarification、logging 等）。

## 架构分析

### 循环委托模型

核心循环逻辑完全由 LangGraph 提供：model call → tool calls → model call 的标准 ReACT 流程。DeerFlow 的角色是在这个循环**外部**包裹中间件管道，形成"核心循环 + 外层装饰器"的两层结构。

### 中间件管道

共 14 个中间件，三类钩子的执行顺序：

- **`before_*` 钩子**：按中间件在 `_build_middlewares` 列表中的顺序**正向**执行（index 0 → 13）
- **`after_*` 钩子**：按**逆序**执行（index 13 → 0），形成洋葱模型
- **`wrap_tool_call`**：包裹每次工具调用，中间件可在工具执行前后各插入逻辑

中间件在 `_build_middlewares()` 中以列表形式静态定义，顺序即优先级。

### Loop Detection 中间件

`LoopDetectionMiddleware` 是 DeerFlow 独有的循环检测机制：

- 维护一个**滑动窗口（20 步）**，对每步的工具调用集合做哈希
- 若同一哈希在窗口内出现 **3 次**，触发警告（写入 messages）
- 出现 **5 次**，强制中断循环

检测粒度是"工具调用集合"而非单个工具调用，能识别 agent 在多工具组合上的卡壳模式。

### Clarification 中间件

`ClarificationMiddleware` 允许在 agent 循环执行中途暂停，向用户请求澄清：

- 通过 `before_agent` 钩子检测模型是否生成了 clarification 意图信号
- 一旦检测到，挂起当前循环，将问题推送给用户
- 用户回复后恢复循环，答案作为新 message 注入 state

这实现了循环内的 human-in-the-loop，而非仅在循环开始/结束时交互。

### 关键代码路径

- `deerflow/agents/lead_agent/agent.py` — `_build_middlewares()` 定义中间件列表及初始化顺序
- `deerflow/agents/middlewares/loop_detection_middleware.py` — 滑动窗口哈希、警告/强制中断逻辑
- `deerflow/agents/middlewares/clarification_middleware.py` — 循环内暂停与用户交互恢复

## 设计亮点

- **中间件管道的可扩展性**：添加新的横切关注点只需新增中间件并插入列表，核心循环代码零修改，符合开闭原则
- **Loop Detection 唯一性**：在已分析的 AI harness 中（Claude Code、OpenHarness），均无类似机制；DeerFlow 是首个将循环检测作为一等功能内置的框架
- **循环内 clarification**：传统 human-in-the-loop 发生在 turn 边界，DeerFlow 允许在单个 turn 的执行中途中断请求澄清，大幅降低 agent 因信息不足而走弯路的概率
- **洋葱模型钩子顺序**：before 正向 + after 逆向的对称结构，使每个中间件可以"拥有"自己的执行上下文，类似 HTTP 中间件的经典设计

## 局限性

- **循环控制权有限**：核心循环由 LangGraph 托管，DeerFlow 无法控制低层级循环机制（如自定义 token streaming 策略、step-level backtracking），灵活性低于完全自实现循环的框架
- **中间件顺序隐性**：优先级由 `_build_middlewares` 列表下标隐性决定，无显式优先级声明，顺序调整容易引入难以排查的行为变化
- **无推理深度控制**：循环中没有动态调整 effort/passes 的机制（OpenHarness 有），所有任务消耗相同的推理资源
- **状态依赖隐式传递**：中间件通过共享 `ThreadState` 通信，缺乏明确的中间件间接口契约，状态耦合风险随中间件数量增长

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
