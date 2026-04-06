---
title: Session Recovery
aliases: [会话恢复, checkpoint, fault tolerance]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[runtime-state]]"
    type: uses
  - target: "[[query-loop]]"
    type: extends
sources: [claude-code]
---

## 一句话定义

Agent 断了怎么办 — checkpoint 保存、状态恢复、容错机制。

## 核心问题

- Checkpoint 保存什么（全部状态 vs 增量）？
- 恢复时怎么判断从哪里继续？
- 网络断开、进程崩溃、用户中断分别怎么处理？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | 恢复"工作现场"而非重放聊天记录：将 file history、attribution state、context collapse 状态、worktree session、agent type、cost state 等完整 runtime state 持久化，跨进程重启后从中断点继续工作 |
| 关键特点 | 交互模式与 headless 模式共享同一套恢复逻辑（`processResumedConversation` 统一协调）；worktree 恢复用 `process.chdir()` 做 TOCTOU-safe 存在性检查；fork session 时提前 seed content-replacement 记录防止 tool_use_id 匹配失败 |
| 局限 | context collapse 恢复依赖 feature flag；coordinator 模式不匹配只报 warning 不强制中断；worktree 被删除后静默降级不告知用户 |

## 设计权衡

- **完整 runtime state 恢复 vs 仅恢复消息历史**：Claude Code 选择了完整状态恢复，将 worktree 路径、git attribution、cost state 等全部纳入持久化范围，因为 agent 在复杂任务中的"工作现场"远不止对话内容。
- **cost state 跨 session 连续 vs 每次重启归零**：cost tracking 不因进程重启而重置，确保 `maxBudgetUsd` 等预算控制在长任务中持续有效，代价是需要在每次 session switch 时执行额外的状态恢复操作。

## L2 详情

- [[session-recovery--claude-code]]
