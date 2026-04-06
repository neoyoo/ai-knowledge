---
title: Tool System
aliases: [工具系统, tool dispatch, function calling, tool use]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[mcp-skills]]"
    type: extends
  - target: "[[query-loop]]"
    type: feeds
sources: [claude-code]
---

## 一句话定义

Agent 运行时中负责发现、选择、调用和管理外部工具的子系统。

## 核心问题

- 怎么让模型知道有哪些工具可用？
- 怎么把模型的意图转成实际的工具调用？
- 怎么处理权限和安全？
- 工具调用失败了怎么处理？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | 完整的工具执行运行时（Tool Execution Runtime）：工具协议对象 + 分层工具池 + 批处理/流式双引擎，权限控制深度嵌入形成"模型可见性 + OS 级沙箱"双层防线 |
| 关键特点 | Fail-closed 并发安全（`isConcurrencySafe` 默认串行）；流式预启动（模型生成时提前启动工具）；三层工具池分离"存在/可见/可执行"三个边界 |
| 局限 | MCP 工具的并发安全声明依赖外部 server，可靠性低于内建工具；自动审批分类器决策逻辑对用户不可见 |

## 设计权衡

- **Fail-closed vs Fail-open 并发**：Claude Code 选择了 fail-closed——工具并发需显式声明 `isConcurrencySafe`，未声明默认串行。牺牲了部分并发性能，但从根本上避免了并发污染导致的状态不一致。
- **工具作为一等实体 vs 裸函数**：每个工具携带 input/output schema、并发安全声明、中断行为声明和权限钩子，使安全语义成为运行时一等成员，而非事后加装。

## L2 详情

- [[tool-system--claude-code]]
