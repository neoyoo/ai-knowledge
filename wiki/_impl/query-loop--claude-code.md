---
title: "Query Loop — Claude Code"
category: L2
parent: "[[query-loop]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的推理主循环由 `query.ts` 和 `QueryEngine.ts` 共同构成，将单次 LLM 调用升级为可恢复、可压缩、可继续的 agent turn 状态机。`query.ts` 作为单回合执行器，负责上下文组装、流式输出消费、工具执行、结果回灌和异常恢复的完整内循环；`QueryEngine.ts` 则作为会话级控制器，封装 SDK 接口、持久化状态和 usage 统计。两者分层协作，形成了 REPL 交互模式和 SDK headless 模式共享的同一套 query runtime。

## 架构分析

### 双核分工：单轮执行器 vs 会话级控制器

**`query.ts` — 单回合执行器**

暴露的核心函数是 `query()`，实现为 async generator，真正的执行逻辑收敛在 `queryLoop()` 内部状态机中。它维护以下显式状态：

- `messages` — 当前对话消息数组，工具结果通过此数组回灌
- `toolUseContext` — 工具执行上下文，跨工具调用共享
- `autoCompactTracking` — 自动压缩触发追踪，包含 token 阈值判断
- `maxOutputTokensRecoveryCount` — max_output_tokens 命中时的恢复计数
- `pendingToolUseSummary` — 待合并的工具结果摘要
- `turnCount` — 轮次计数，用于循环终结判断
- `transition` — 状态机跳转信号

这些显式状态使"一轮推理"成为可以被观测、中断和恢复的对象，而不是一次性的异步调用。

**`QueryEngine.ts` — 会话级控制器**

将 `query()` 封装为面向 SDK 和 headless 场景的类，持有：

- `mutableMessages` — 跨多轮持久化的消息数组
- `abortController` — 中止控制，支持用户手动取消
- 累积 usage 统计（token 消耗、工具调用次数）
- read file cache — 减少重复文件读取
- permission denial 记录 — 追踪被拒绝的工具调用
- 用户输入 → `SDKMessage` 的格式转换

`QueryEngine` 的存在让 REPL 和 SDK 两种使用模式共享同一套 query runtime，避免了两套 agent 实现的分叉。

### 单轮执行的标准路径

一次 agentic turn 的标准执行顺序：

```
1. 构建 system prompt + user context + system context
2. 预估 token 与 budget（applyToolResultBudget）
3. 判断是否触发 auto compact 或 reactive compact
4. 向模型发起流式请求
5. 消费 assistant stream，收集 text block 和 tool_use block
6. 并发执行收到的工具调用
7. 将 tool_result 写回 messages 数组
8. 判断终结条件：无工具调用 / stop reason = end_turn / 轮次超限
9. 如未终结，继续下一次 queryLoop 迭代
10. 输出 terminal state 给上层
```

关键特征：工具执行完毕后结果立即回灌，模型可以在同一 turn 内持续推进，直到自主停止或触发终结条件。这是 agentic loop 区别于单次问答的核心机制。

### Thinking Block 的协议化处理

`query.ts` 对 thinking block（扩展思考内容）有专门的合法性规则：

- thinking block 只能在支持扩展思考的 query 中出现
- thinking block 不能作为 assistant trajectory 的最后一个 block
- 在消息历史中必须维持结构合法（不能孤立存在）

这说明 Claude Code 把模型思考过程当作结构化协议对象处理，而不是普通文本。错误的 thinking block 排列会被检测并修正，以确保模型收到的历史消息在结构上始终合法。

### Tool Result Budget 控制

`applyToolResultBudget()` 在工具结果被写入消息流之前控制其体积。这解决了工具返回内容过大导致上下文爆炸的问题——系统不是让大工具结果直接进入 context，而是在写入前做截断或摘要处理。

这表明 Claude Code 把"工具返回太大"视为一类一等运行时问题，而不是交给模型自行处理。

### Max Output Tokens Recovery 机制

当模型命中 `max_output_tokens` 上限时，系统不直接失败，而是：

1. 递增 `maxOutputTokensRecoveryCount`
2. 将中间错误暂时 withheld（不暴露给 SDK 调用方）
3. 尝试以更小的 output token 预算重试
4. 超过恢复次数上限才向上层传播错误

这个机制将一类常见的基础设施限制变成可自动恢复的瞬时故障，提升了 agent 在长任务中的鲁棒性。

### Stop Hooks 与 Post Hooks 插槽

主循环在标准 model → tool → model 路径之外，还为 stop hooks 和 post hooks 提供了明确插槽。这意味着：

- 用户可以在每轮结束后插入自定义逻辑（如格式校验、日志记录）
- 平台治理逻辑（如行为审计）可以在不修改核心循环的情况下注入

这是关注点分离的设计——核心执行路径保持干净，扩展需求通过 hook 插槽注入。

### Auto Compact 集成

`autoCompactTracking` 状态在每轮开始时检查当前 token 使用量，判断是否触发自动压缩：

- **auto compact**：在循环开始前预判，提前触发
- **reactive compact**：在工具执行后 token 超限时触发

两种触发路径都在 `queryLoop` 内部处理，压缩后继续循环，对上层调用方透明。

### 关键代码路径

- `src/query.ts` — `query()` 入口，`queryLoop()` 内部状态机，thinking block 规则，max_output_tokens recovery
- `src/QueryEngine.ts` — 会话级控制器，SDK 封装，mutableMessages，usage 统计
- `src/services/tools/toolOrchestration.ts` — 工具调用并发编排
- `src/services/tools/StreamingToolExecutor.ts` — 流式工具执行器
- `src/services/compact/autoCompact.ts` — 自动压缩触发逻辑
- `src/services/compact/compact.ts` — 压缩执行实现
- `src/services/SessionMemory/sessionMemory.ts` — 记忆系统集成点

## 设计亮点

- **Agent turn 显式建模为状态机**：`queryLoop` 的显式状态（messages、toolUseContext、autoCompactTracking 等）让 agent 执行过程可被观测和中断，不再是"发送 → 等待 → 返回"的黑盒。这是 agent loop 工程化的正确方向。
- **Max output tokens 的优雅降级**：把基础设施限制（token 上限命中）内化为可自恢复的瞬时故障，而不是让上层业务处理，体现了稳健性优先的设计哲学。
- **Tool result budget 的主动控制**：在工具结果写入 context 前做体积控制，将上下文管理责任前移到执行层，而不是事后被动压缩，更精确也更高效。
- **REPL 与 SDK 共享同一 runtime**：`QueryEngine` 的封装层设计让两种使用模式不维护各自的 agent 实现，降低了系统的整体复杂度，任何 core loop 的改进自动惠及两种模式。

## 局限性

- **`queryLoop` 状态机缺乏形式化描述**：当前状态转换逻辑散落在函数体中，没有显式的状态转换图或类型，复杂 case（如 recovery 后再次命中 compact）的行为依赖阅读源码才能理解。
- **工具并发度缺乏动态控制**：工具调用目前的并发策略在 `toolOrchestration.ts` 中处理，但没有基于系统负载或工具类型动态调整并发度的机制，在工具密集型任务中可能产生资源竞争。
- **Turn count 终结条件较粗**：`turnCount` 超限是终结主循环的条件之一，但固定轮次上限对于不同复杂度的任务是相同的，不够自适应。
- **Recovery 机制的上限次数硬编码**：`maxOutputTokensRecoveryCount` 的恢复上限目前是固定值，不随任务类型或上下文大小调整，极端场景下可能过早放弃。

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
