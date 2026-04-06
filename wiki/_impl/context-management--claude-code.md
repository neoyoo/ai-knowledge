---
title: "Context Management — Claude Code"
category: L2
parent: "[[context-management]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 把上下文爆炸视为主路径问题（primary path），而不是边缘异常。其上下文管理系统是一个主动调度器：持续监控 token 使用、提前保留 headroom、在阈值触发时执行压缩、并在失败时熔断。压缩的目标不是把历史"缩写成一段话"，而是生成一个能继续工具调用、能继续推理的完整对话投影（context projection），从而让 agent 无缝衔接压缩前后的状态。

## 架构分析

### 主动调度：autoCompact.ts 的角色

`src/services/compact/autoCompact.ts` 是压缩的调度层，而非算法本体。它的核心职责是判断"何时该压缩"，具体包括：

- **`getEffectiveContextWindowSize()`**：计算有效上下文窗口。不是总窗口大小，而是总窗口减去为输出和摘要预留的空间（headroom）
- **阈值计算**：分别计算 auto-compact 触发阈值、warning 阈值、error 阈值、blocking limit
- **`shouldCompact()`**：综合当前 token 使用与各阈值，决定是否触发压缩

这一设计的关键在于：**不是"超限后救火"，而是"提前保留空间"**。系统在上下文窗口被吃满之前就主动介入。

### 熔断机制：MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES

`autoCompact.ts` 中存在连续压缩失败计数器。当压缩连续失败次数达到上限时，系统停止自动触发压缩。

这一设计的背景是：某些超长上下文在结构上无法被压缩（例如工具调用嵌套过深、消息格式不合法等），盲目重试只会造成每轮 agent loop 都卡在注定失败的压缩操作上。熔断机制把压缩视为**可能失败的 runtime 行为**，而非理所当然的成功路径。

### 压缩算法：compact.ts 的目标

`src/services/compact/compact.ts` 负责真正构造压缩后的消息结构。与简单摘要器不同，它的输出必须满足：

- **可继续推理**：压缩后的上下文仍能支持后续的多步推理
- **可继续工具调用**：工具 use/result 的结构语义必须在压缩后保持一致
- **可恢复状态**：agent 能从压缩后的快照继续工作，而非从头开始

这意味着压缩结果不是自由文本摘要，而是一个符合对话协议的"状态快照"。

### 模式冲突规避

系统在某些模式下会主动抑制自动压缩，例如：

- `session_memory` 模式：此时 memory 系统负责长期状态管理，压缩和 memory 系统同时运行会导致状态双写或覆盖
- `compact` 模式本身（防止递归触发）
- context-collapse 相关场景

这体现了一个重要的设计原则：**多个上下文管理机制之间需要协调，否则会互相覆盖或产生矛盾**。

### 与 query.ts 的集成

`src/query.ts` 是 agent loop 的主入口，autoCompact 在每轮循环中被调用。这使上下文管理成为 loop 的内嵌组件，而非外挂模块。

### 关键代码路径

- `src/services/compact/autoCompact.ts` — 压缩调度器（阈值计算、有效窗口、熔断、触发判断）
- `src/services/compact/compact.ts` — 压缩算法本体（构造可继续执行的对话快照）
- `src/query.ts` — agent loop 主入口，autoCompact 在此被调用

## 设计亮点

- **Headroom 预留**：`getEffectiveContextWindowSize()` 把输出空间从可用窗口中提前扣除，避免"写满写不下"的死锁情况
- **熔断而非无限重试**：对于结构性不可压缩的上下文，系统选择放弃自动压缩而非死循环，体现了对真实失败模式的深刻理解
- **压缩即状态快照**：compact.ts 的输出是符合对话协议的结构化快照，而不是自由文本摘要；这使得压缩后 agent 能无缝继续工作，不出现"失忆断层"
- **模式感知**：上下文系统知道自己不是孤立运行的，在 session_memory 等模式下主动退出，避免多机制互相覆盖

## 局限性

- **压缩质量依赖模型**：compact.ts 调用的是 LLM 来生成压缩快照，压缩质量受模型能力约束，且可能引入额外 token 消耗和延迟
- **熔断后无降级策略**：连续失败超限后，系统停止自动压缩，但没有进一步的降级手段（如截断最旧消息）；用户此时可能遭遇硬性 context limit 错误
- **阈值为静态配置**：auto-compact 触发阈值在系统内是固定计算逻辑，无法根据任务类型或历史压缩成功率动态调整

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
