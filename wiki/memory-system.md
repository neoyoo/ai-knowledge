---
title: Memory System
aliases: [记忆系统, persistent memory, cross-session memory]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[context-management]]"
    type: feeds
  - target: "[[runtime-state]]"
    type: uses
sources: [claude-code]
---

## 一句话定义

跨会话的持久记忆 — 和 context 不同，活过对话结束，下次还在。

## 核心问题

- 什么值得记住，什么不值得？
- 记忆怎么存储（文件/数据库/向量）？
- 怎么在新对话开始时检索相关记忆？
- 记忆过时了怎么处理？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | 在上下文压缩之外引入 Session Memory File（L3 知识载体），将长期稳定信息剥离为可落盘、可审阅的 Markdown 工作笔记，由后台 subagent 异步提取，不阻塞主会话 |
| 关键特点 | 多维触发保护（`shouldExtractMemory()` 综合 token 规模/增量/tool call 次数/自然停顿点）；文件即记忆（人工可读、可编辑、可版本控制）；三层知识载体明确分工（消息历史/压缩快照/Session Memory） |
| 局限 | 记忆内容质量依赖 LLM 归纳；跨会话 memory file 不自动合并；后台提取无用户可见反馈 |

## 设计权衡

- **隐式黑盒状态 vs 文件化透明记忆**：Claude Code 选择了文件化——Session Memory 落盘为 Markdown，用户可打开、审阅、手动编辑。牺牲了内部状态的封装性，换取了完整的可解释性和可干预性。
- **同步提取（阻塞交互） vs 异步后台提取**：选择 forked subagent 后台提取，主会话交互不受影响，但代价是提取失败时用户无感知，需要接受一定的不可靠性。

## L2 详情

- [[memory-system--claude-code]]
