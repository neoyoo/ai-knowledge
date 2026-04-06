---
title: Runtime State
aliases: [运行时状态, state management, agent state]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[session-recovery]]"
    type: feeds
sources: [claude-code]
---

## 一句话定义

Agent 运行时的状态容器 — 管理当前会话信息、配置、运行模式和生命周期。

## 核心问题

- 哪些状态是全局的，哪些是单轮的？
- 状态怎么序列化（给 checkpoint/recovery 用）？
- 多 agent 场景下状态怎么隔离？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | `AppStateStore`（统一状态树）+ `REPL.tsx`（交互运行时协调器）：前者承载 agent 工作台完整状态（权限/任务/MCP/IDE），后者串联 prompt 构建、query 驱动、工具审批 UI 和 hooks 生命周期 |
| 关键特点 | REPL 不是薄渲染层，而是 prompt 构建参与者 + 工具审批 UI 持有者；工具审批 UI 与权限记录和 prompt 规则形成完整闭环；通过 async generator yield 机制解耦 REPL 与 query.ts |
| 局限 | AppStateStore 边界不清，新功能容易随意挂载导致状态树膨胀；REPL.tsx 职责过重，局部修改影响面难以评估 |

## 设计权衡

- **单一全局状态树 vs 分散状态管理**：Claude Code 选择了单一统一状态树，让 agent 工作台的跨组件协调成本极低——但代价是状态树随功能增加持续膨胀，边界越来越难维护。
- **REPL 作为编排器 vs 薄渲染层**：把 prompt 构建、工具审批、任务管理等职责集中在 REPL 中，保持了"交互域"与"认知域"（query.ts）的清晰分工，但也让 REPL.tsx 成为最重的单一文件。

## L2 详情

- [[runtime-state--claude-code]]
