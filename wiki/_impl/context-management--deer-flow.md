---
title: "Context Management — DeerFlow"
category: L2
parent: "[[context-management]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 的上下文管理围绕 `SummarizationMiddleware`（来自 LangChain）实现自动压缩。触发条件高度可配置：可以按 token 数、消息数或占最大上下文的比例三种方式任选，触发后保留最近 N 条消息，旧消息替换为摘要。Token 计数使用 tiktoken 精确计算。中间件在 `after_model` 钩子上挂载，每次模型响应后自动检查是否需要压缩，压缩逻辑与对话循环完全解耦。可选配更轻量的专用摘要模型以降低成本。

## 架构分析

### 触发策略（三选一）

通过 `summarization_config.py` 配置触发条件，支持三种模式：

- **token_count**：当对话总 token 数超过阈值时触发；
- **message_count**：当消息条数超过阈值时触发；
- **fraction**：当已用 token 占模型最大上下文的比例超过阈值时触发（如 0.75）。

`keep` 参数控制触发后保留最近多少条消息不被摘要，确保短期上下文连贯性。

### 压缩执行流程

1. `after_model` 钩子在每次模型返回后调用 `SummarizationMiddleware.check()`；
2. 若触发条件满足，提取 `messages[:-keep]` 作为待压缩部分；
3. 调用摘要模型（可配置为更小的模型）生成摘要文本；
4. 用单条摘要消息替换原有旧消息，`messages[-keep:]` 原样保留；
5. 更新后的消息列表继续传入下一轮对话。

### Token 精确计数

使用 tiktoken 库对每条消息编码后计数，避免字符数估算误差。这使得基于 token 的触发条件和 fraction 模式都能精确工作，不会因模型换代导致计数偏差。

### 关键代码路径

- `deerflow/config/summarization_config.py` — 触发条件、keep 参数、摘要模型配置
- `deerflow/agents/middlewares/` — SummarizationMiddleware 注册与 after_model 钩子挂载

## 设计亮点

- **三模式触发最灵活**：token_count/message_count/fraction 三种条件覆盖不同部署场景——嵌入式场景用 message_count、API 成本敏感场景用 fraction，是目前所有对比项目中触发策略最灵活的；
- **摘要模型可独立配置**：可以用 gpt-4o-mini 或更小的模型做摘要，主模型用强模型推理，成本结构更优；
- **中间件模式干净解耦**：压缩逻辑完全在 middleware 层，不侵入 agent loop 和模型调用路径，替换或关闭都不影响其他组件。

## 局限性

- **依赖 LangChain 内置实现**：摘要质量和策略受 LangChain SummarizationMiddleware 约束，无法像 Claude Code 自定义压缩提示词那样精细控制摘要内容；
- **无熔断机制**：摘要调用失败时没有降级处理（Claude Code 有 circuit breaker），若摘要模型不可用会阻塞对话；
- **压缩不感知语义边界**：按消息数切割，不考虑工具调用链或任务边界，可能在语义中间截断，影响摘要质量。

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
