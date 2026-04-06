---
title: "Context Management — OpenHarness"
category: L2
parent: "[[context-management]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness 的上下文管理以极简为核心设计理念：token 估算用字符数除以 4 的启发式公式，压缩逻辑仅 58 行，通过滑动窗口保留最近 N 条消息、将旧消息替换为单条拼接文本摘要。无自动触发压缩，无调用模型生成摘要，成本追踪通过 `CostTracker` 独立维护。整套实现刻意回避 SDK 依赖和异步 tokenizer，以可读性和零依赖换取精度牺牲。

## 架构分析

### Token 估算

`token_estimation.py` 使用公式 `max(1, (len(text)+3)//4)` 将字符数映射为 token 估算值。这是对英文平均 token 长度约 4 字符的粗粒度近似，对中文、代码等场景误差显著。优点是无网络调用、无模型依赖、确定性计算，适合快速判断是否接近阈值。

### 消息压缩（compact_messages）

`compact_messages()` 实现滑动窗口压缩：将 `messages[:-preserve_recent]`（默认保留最近 6 条）的所有消息文本拼接为一个字符串，替换为单条合成的 assistant 消息。没有调用语言模型生成摘要——压缩内容即原始文本的字符串拼接，信息无损但格式扁平化。压缩后 token 计数降低是因为旧消息的结构开销（role 字段、列表项）被消除，而非内容被压缩。

### 触发机制

自动压缩功能存在但**必须显式调用**，不会在 token 阈值被突破时自动触发。这与 Claude Code 等系统的隐式自动触发形成对比。调用方需要在 agent loop 中主动检查 token 估算值并决定何时压缩。

### 成本追踪

`CostTracker` 通过 `UsageSnapshot`（包含 `input_tokens` + `output_tokens`）累积每次 API 调用的 token 消耗。追踪与压缩逻辑解耦，`CostTracker` 仅做累加，不参与压缩决策。

### 关键代码路径

- `services/compact/__init__.py` — `compact_messages()` 全部实现，共 58 行
- `services/token_estimation.py` — 字符数/4 启发式 token 估算函数
- `engine/cost_tracker.py` — `CostTracker` 与 `UsageSnapshot` 定义，token 消耗累积

## 设计亮点

- 整个压缩子系统 58 行完成，代码可在 5 分钟内完全理解，极低的维护负担
- 零外部依赖：无 tiktoken、无 tokenizer 模型、无异步初始化，冷启动即可用
- 成本追踪与压缩逻辑完全解耦，各自职责清晰，独立可测试
- 字符数估算足以支撑"是否需要关注"的粗粒度判断，避免过度工程化

## 局限性

- 字符数/4 启发式对中文、代码、特殊符号误差可达 2-5 倍，无法用于精确 token 预算管理
- 压缩不自动触发，agent loop 需手动检测阈值并调用，增加使用方复杂度
- `compact_messages()` 只做文本拼接而非语义摘要，压缩后旧上下文可读性差，模型难以利用
- UI 层无 token budget 显示，用户无法感知上下文压力
- 无压缩失败的熔断机制，若压缩后 token 仍超限（如单条消息极长），无降级策略

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
