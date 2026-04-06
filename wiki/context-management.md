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
sources: [claude-code, openharness]
---

## 一句话定义

管理有限的 context window — 什么放进去、什么压缩、什么丢掉。

## 核心问题

- 当对话超长时，怎么决定保留哪些信息？
- 压缩策略：截断 vs 摘要 vs 向量检索？
- Token 预算怎么分配给不同模块（系统指令/历史/工具结果）？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | 主动调度器而非被动救火：持续监控 token 使用、提前保留 headroom、阈值触发时执行压缩，输出可继续推理和工具调用的完整对话快照（context projection） | 极简设计：token 估算用字符数/4 的启发式公式，压缩逻辑仅 58 行，通过滑动窗口保留最近 N 条消息、将旧消息替换为单条拼接文本摘要；无自动触发，无调用模型生成摘要 |
| 关键特点 | `getEffectiveContextWindowSize()` 提前扣除输出预留空间；连续失败熔断机制（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）；模式感知——session_memory 模式下主动抑制自动压缩 | 整个压缩子系统 58 行完成，零外部依赖（无 tiktoken、无异步初始化）；成本追踪与压缩逻辑完全解耦；字符数估算足以支撑粗粒度判断，避免过度工程化 |
| 局限 | 压缩质量依赖 LLM 能力；熔断后无降级策略（无截断最旧消息等回退手段）；触发阈值为静态配置不可动态调整 | 字符数/4 对中文、代码误差可达 2-5 倍；压缩不自动触发，需 agent loop 手动检测阈值；`compact_messages()` 只做文本拼接而非语义摘要，旧上下文可读性差 |

## 设计权衡

- **救火式压缩 vs 主动 headroom 预留**：Claude Code 选择了主动调度——在窗口被吃满之前就介入，因为被动压缩（超限才触发）会造成输出无空间的死锁，而预留 headroom 能确保模型始终有输出能力。
- **自由文本摘要 vs 结构化状态快照**：压缩输出是符合对话协议的"状态快照"而非自由文本，因为 agent 必须能从压缩后继续工具调用和多步推理，不允许出现"失忆断层"。

## L2 详情

- [[context-management--claude-code]]
- [[context-management--openharness]]
