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
sources: []
---

## 一句话定义

Agent 断了怎么办 — checkpoint 保存、状态恢复、容错机制。

## 核心问题

- Checkpoint 保存什么（全部状态 vs 增量）？
- 恢复时怎么判断从哪里继续？
- 网络断开、进程崩溃、用户中断分别怎么处理？

## 各家对比

| 维度 | （待填充） |
|------|-----------|

## 设计权衡

（待填充）

## L2 详情

（待导入）
