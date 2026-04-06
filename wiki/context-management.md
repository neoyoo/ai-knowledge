---
title: Context Management
aliases: [上下文管理, context window, token budgeting]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[prompt-system]]"
    type: feeds
  - target: "[[memory-system]]"
    type: uses
sources: []
---

## 一句话定义

管理有限的 context window — 什么放进去、什么压缩、什么丢掉。

## 核心问题

- 当对话超长时，怎么决定保留哪些信息？
- 压缩策略：截断 vs 摘要 vs 向量检索？
- Token 预算怎么分配给不同模块（系统指令/历史/工具结果）？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
