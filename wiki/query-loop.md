---
title: Query Loop
aliases: [agent loop, 主循环, agentic loop]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[tool-system]]"
    type: uses
  - target: "[[prompt-system]]"
    type: uses
  - target: "[[context-management]]"
    type: uses
sources: [claude-code]
---

## 一句话定义

Agent 的主循环 — 发请求给模型、拿结果、判断下一步（继续/调工具/结束），再来一轮。

## 核心问题

- 循环什么时候结束？谁来判断？
- 一轮里面的状态怎么传递？
- 出错了怎么处理（重试/降级/终止）？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | `query.ts` + `QueryEngine.ts` 双核分工：单回合执行器（状态机）+ 会话级控制器，将 agent turn 显式建模为可观测、可中断、可恢复的状态机 |
| 关键特点 | max_output_tokens 命中后自动降级重试（不直接失败）；工具结果写入前做 budget 控制（`applyToolResultBudget`）；REPL 与 SDK headless 模式共享同一 runtime |
| 局限 | 状态转换逻辑散落函数体中，无形式化状态转换图；固定 turnCount 上限不自适应任务复杂度 |

## 设计权衡

- **黑盒调用 vs 显式状态机**：Claude Code 选择了显式状态机（维护 messages、toolUseContext、autoCompactTracking 等显式状态），因为 agent 执行过程需要可观测、可中断和可恢复，而不仅仅是"发送并等待"。
- **主动 budget 控制 vs 被动压缩**：工具结果在写入 context 前做体积控制（`applyToolResultBudget`），把上下文管理责任前移到执行层，比事后被动压缩更精确高效。

## L2 详情

- [[query-loop--claude-code]]
