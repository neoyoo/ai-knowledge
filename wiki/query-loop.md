---
title: Query Loop
aliases: [agent loop, 主循环, agentic loop]
category: L1
created: 2026-04-06
updated: 2026-04-08
relations:
  - target: "[[tool-system]]"
    type: uses
  - target: "[[prompt-system]]"
    type: uses
  - target: "[[context-management]]"
    type: uses
sources: [claude-code, openharness, deer-flow, hermes-agent]
---

## 一句话定义

Agent 的主循环 — 发请求给模型、拿结果、判断下一步（继续/调工具/结束），再来一轮。

## 核心问题

- 循环什么时候结束？谁来判断？
- 一轮里面的状态怎么传递？
- 出错了怎么处理（重试/降级/终止）？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|-------------|
| 核心设计 | `query.ts` + `QueryEngine.ts` 双核分工：单回合执行器（状态机）+ 会话级控制器，将 agent turn 显式建模为可观测、可中断、可恢复的状态机 | `run_query()` 为核心，实现为最多 8 轮（可配置）的 `for` 循环：流式接收响应，有工具调用则执行（多工具通过 `asyncio.gather()` 并行），无工具调用则直接返回 | 循环本体委托给 LangGraph 的 `create_agent()`，自身创新集中在 14 个中间件组成的管道上，以 `before_agent`/`after_model`/`wrap_tool_call` 三类钩子介入每个关键节点，在不修改核心循环代码的前提下注入横切关注点 | `run_conversation()` 单一扁平循环（2400 行）：双条件守卫（`api_call_count < max_iterations AND iteration_budget.remaining > 0`）；所有逻辑（API 调用、工具执行、错误恢复、上下文压缩、预算管理）全压入同一方法体，核心哲学是**防御性工程** |
| 关键特点 | max_output_tokens 命中后自动降级重试（不直接失败）；工具结果写入前做 budget 控制（`applyToolResultBudget`）；REPL 与 SDK headless 模式共享同一 runtime | 单/多工具执行路径显式分支，代码意图一目了然；`asyncio.gather()` 并行执行多工具是 first-class 设计；`CostTracker` 内嵌于 `QueryEngine`，费用追踪随对话历史自然流动 | Loop Detection：滑动窗口（20 步）哈希检测，3 次触发警告、5 次强制中断循环；循环内 clarification：允许在 turn 执行中途暂停请求用户澄清；洋葱模型钩子顺序（before 正向 + after 逆向） | 9 个类型化回调（thinking/stream_delta/tool_start/tool_complete/step 等），回调异常不中断循环；`IterationBudget` 线程安全计数器，`execute_code` 工具免费（自动 refund）；路径级并行安全判定（同名工具写不同路径才并行）；预算压力注入工具结果 JSON（不发新消息） |
| 局限 | 状态转换逻辑散落函数体中，无形式化状态转换图；固定 turnCount 上限不自适应任务复杂度 | 超过 `max_turns` 直接抛出 `RuntimeError`，无优雅降级；对话历史不做压缩，长对话可能撑爆 context window | 核心循环由 LangGraph 托管，无法控制低层级循环机制；中间件优先级由列表下标隐性决定；无推理深度控制（无 effort/passes 机制） | 单函数 2400 行，可测试性弱，边缘 case 行为只能读源码；父子 agent 预算独立不共享（90+5×50=340 实际上限远超预期）；工具结果无体积控制层（无 `applyToolResultBudget` 等价物） |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 方案 A：简单循环（Loop-based） | `while True: call_model() → check_tool_use() → execute_tools() → continue_or_break`，线性推进，无显式状态机 | 单用户 CLI agent、编程助手、对话型 agent | Claude Code、OpenHarness、Cursor |
| 方案 B：图编排（Graph-based） | 显式状态机，节点是函数，边是状态转换，工作流可视化 | 多步审批链、数据管道、有复杂分支条件的业务流程 | LangGraph、Temporal AI |
| 方案 C：对话式协作（Conversation-based） | 多个 agent 在共享对话中轮流发言，互相驱动推进 | 需要多视角分析的任务、模拟团队协作、辩论式评审 | AutoGen、ChatDev、MetaGPT |

### 场景决策指南

**如果你在做编程助手或 CLI 工具 → 选方案 A（简单循环）**
- 原因：任务本质是线性的——理解需求、写代码、运行、修错，单线程顺序推理最清晰可调试
- 注意：必须设定 `max_turns` 上限（建议 20-50），不然一旦工具出错或模型陷入循环就是无限烧钱；出错处理（重试/降级/终止）需要在循环内显式处理，不能靠框架自动救火

**如果你在做工作流引擎（审批、数据管道、多阶段 ETL）→ 选方案 B（图编排）**
- 原因：复杂分支逻辑（人工审批节点、条件跳转、并行子任务）用图来表达比代码里堆 `if/else` 和状态标志清晰得多，状态转换有明确边界
- 注意：图编排对简单任务是过度工程化，引入 LangGraph 等框架有学习成本；调试时需要理解图执行引擎本身的行为，出错堆栈比简单循环复杂得多

**如果你在做需要多视角的分析任务（代码审查、方案评估、报告生成）→ 选方案 C（对话式协作）**
- 原因：不同 agent 扮演不同角色（批评者、辩护者、总结者），天然形成多视角对冲，适合需要"红队蓝队"式质量保证的场景
- 注意：token 消耗是单 agent 的数倍；必须设计明确的终止条件（如"达成共识则结束"或"最多 N 轮"），否则 agent 之间容易陷入循环确认；不适合需要确定性输出的任务

### 错误恢复粒度

各项目对错误恢复的处理粒度差异很大，是设计复杂度的核心分水岭：

- **最少路径（OpenHarness）**：超限直接抛 `RuntimeError`，调用方处理——简单但不优雅，适合工具调用失败率低的内部场景
- **中等路径（Claude Code）**：`max_output_tokens` 降级重试 + 工具结果体积控制，覆盖最常见的两类问题
- **完整路径（Hermes Agent）**：10+ 条独立恢复路径——无效工具名模糊修复、JSON 参数重试、截断响应续写、速率限制退避、上下文溢出压缩、思维块签名失效清理、provider 切换…… 每类异常有专属处理逻辑

**决策建议**：面向公网/多 provider 的生产 agent 需要完整路径；内部工具或受控环境只需中等路径；路径越多可测试性越弱，需权衡。

### fallback provider 的作用域设计

Hermes Agent 揭示了一个关键设计问题：**fallback 激活后应该持续多久？**

- **turn-scoped fallback（Hermes 做法）**：每 turn 开始时恢复主模型，fallback 只对当前 turn 生效。好处是"降级是异常而非常态"，主模型始终优先；代价是每 turn 都要探测主模型是否恢复，有额外延迟
- **session-scoped fallback（更常见做法）**：一旦切换 fallback，整个会话都用 fallback 模型。省去探测开销，但可能让用户长期用到次优模型而不自知

**结论**：对话体验敏感（语音、实时交互）的场景用 turn-scoped，批处理/后台任务用 session-scoped 即可。

### 预算压力的注入方式

当迭代次数接近上限时，如何通知模型"快没机会了"？两种思路：

- **注入新消息（user/system 角色）**：直观，但会破坏消息角色结构（assistant → tool → user 交替），部分模型会把 system 注入当作指令干扰正常推理
- **注入工具结果 JSON（Hermes 做法）**：把 `_budget_warning` 字段写进最后一条 tool result，不新增消息轮次；跨 turn 自动清理，防止残留影响下一 turn 行为

这个细节在多 turn 会话中尤其重要——残留的预算警告可能让模型在预算充足的下一轮仍表现得焦虑急躁。

### 并行工具执行的安全粒度

Hermes Agent 将并行工具执行的安全判断细化到**路径级**：同一工具对不同路径可并行，对同一路径必须串行。这比"写操作一律不并行"的粗粒度策略更高效，但实现复杂度更高（需要 `_paths_overlap()` 路径重叠检测）。

**工程决策**：工具执行速度是瓶颈且有高并发需求时，值得投入路径级安全判定；否则"同类型写操作串行"的简单规则已足够。

### 常见陷阱

- **用图编排做简单 QA**：引入 LangGraph 只是为了"问问题、拿答案"，是典型的过度工程化——维护成本增加 10 倍，性能提升为零
- **用简单循环做复杂审批流**：审批涉及"等待人工确认 → 条件分叉 → 并行子任务"，强行用 while 循环实现，状态标志迅速膨胀成难以维护的意大利面条代码
- **忘记设置最大轮次**：模型反复调用工具不收敛（工具失败、输出不符合预期、prompt 指令模糊），没有硬性上限就是开着水龙头不关，千次调用的账单等着你
- **错把对话式协作当并行加速**：多 agent 对话本质是串行的（agent A 发言 → agent B 响应），不会加速执行，只会增加 token 消耗，别期望它提升吞吐量
- **父子 agent 预算不联动**：多 agent 场景下若父子预算各自独立，实际总迭代数可远超单一 max_iterations 的预期（1 parent × 90 + 5 subagents × 50 = 340 次），必须在系统级设计预算上限，而不只靠单个 agent 的 max_iterations

## L2 详情

- [[query-loop--claude-code]]
- [[query-loop--openharness]]
- [[query-loop--deer-flow]]
- [[query-loop--hermes-agent]]
