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
sources: [claude-code, openharness]
---

## 一句话定义

跨会话的持久记忆 — 和 context 不同，活过对话结束，下次还在。

## 核心问题

- 什么值得记住，什么不值得？
- 记忆怎么存储（文件/数据库/向量）？
- 怎么在新对话开始时检索相关记忆？
- 记忆过时了怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | 在上下文压缩之外引入 Session Memory File（L3 知识载体），将长期稳定信息剥离为可落盘、可审阅的 Markdown 工作笔记，由后台 subagent 异步提取，不阻塞主会话 | 忠实移植 Claude Code 记忆模式：以 `~/.openharness/data/memory/{project-name}-{sha1}/` 为存储根，`MEMORY.md` 作为索引入口，prompt 构建时注入全文并通过纯词法匹配（约 20 行）选取最相关的最多 5 个专题文件注入 |
| 关键特点 | 多维触发保护（`shouldExtractMemory()` 综合 token 规模/增量/tool call 次数/自然停顿点）；文件即记忆（人工可读、可编辑、可版本控制）；三层知识载体明确分工（消息历史/压缩快照/Session Memory） | 词法检索完全无外部依赖，无需 embedding 模型或向量数据库；文件系统存储人类可读、可直接编辑、可用 git 版本控制；首行元数据约定（160 字符描述）将检索开销降到极低 |
| 局限 | 记忆内容质量依赖 LLM 归纳；跨会话 memory file 不自动合并；后台提取无用户可见反馈 | 检索仅匹配标题和首行描述，正文内容对检索不可见；无后台自动提取机制，记忆写入完全依赖显式调用；无跨项目记忆共享能力 |

## 设计权衡

- **隐式黑盒状态 vs 文件化透明记忆**：Claude Code 选择了文件化——Session Memory 落盘为 Markdown，用户可打开、审阅、手动编辑。牺牲了内部状态的封装性，换取了完整的可解释性和可干预性。
- **同步提取（阻塞交互） vs 异步后台提取**：选择 forked subagent 后台提取，主会话交互不受影响，但代价是提取失败时用户无感知，需要接受一定的不可靠性。

## L2 详情

- [[memory-system--claude-code]]
- [[memory-system--openharness]]
