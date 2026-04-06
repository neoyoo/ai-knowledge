---
title: "Evaluation Observability — OpenHarness"
category: L2
parent: "[[evaluation-observability]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness 的可观测性层围绕 token 计数和结构化事件流两个支柱构建：`CostTracker` 跨 turn 累计 usage 统计，`stream-json` 输出模式以换行分隔 JSON 事件流暴露实时执行过程，供外部工具消费。整体能力覆盖基础监控需求，但与生产级可观测性体系存在显著差距。

## 架构分析

### Token 计量

API 客户端从 `final_message.usage` 提取 `input_tokens` / `output_tokens`，封装为 `UsageSnapshot` 值对象。`CostTracker` 持有跨 turn 的累计状态，在每次 API 响应后更新。当前仅记录 token 数量，不关联定价表，无法换算为美元成本。

### stream-json 事件流

通过 `oh -p "..." --output-format stream-json` 激活。程序以换行分隔 JSON（NDJSON）格式向 stdout 输出以下事件类型：

- `assistant_delta` — 流式 token 增量
- `tool_started` — 工具调用开始（含工具名和输入参数）
- `tool_completed` — 工具调用完成（含输出结果）
- `assistant_complete` — 当前 turn 的完整 assistant 消息

这是 OpenHarness 唯一的机器可读可观测接口，设计上对齐外部工具链集成（如管道到 `jq`、日志收集器、CI 脚本）。

### 调试日志

通过 Python 标准 `logging` 模块输出，无结构化格式。调试模式下可观察 API 请求/响应细节，但不适合程序化消费。

### 关键代码路径

- `openharness/engine/cost_tracker.py` — `CostTracker` 类，跨 turn 累计 token 统计
- `openharness/api/usage.py` — `UsageSnapshot` 值对象，从 API 响应提取 usage 字段
- `openharness/engine/stream_events.py` — 事件类型定义（`assistant_delta` / `tool_started` / `tool_completed` / `assistant_complete`）
- `openharness/ui/app.py` — `stream-json` 输出模式实现，NDJSON 序列化与 stdout 写入

## 设计亮点

- **clean machine-readable interface**：`stream-json` 模式提供结构清晰的事件流，外部工具可直接 pipe 消费，无需解析 ANSI 终端输出
- **事件粒度合理**：tool_started/tool_completed 配对结构便于计算工具调用延迟和成功率
- **零依赖**：不引入 OpenTelemetry SDK 等重型依赖，保持运行时轻量

## 局限性

- **最显著的能力缺口**：与 Claude Code 相比，可观测性是 OpenHarness 差距最大的维度
- **无定价换算**：仅 token 数量，无美元成本估算，无分模型定价表
- **无结构化遥测**：未集成 OpenTelemetry，无 trace/span 概念，无分布式追踪能力
- **无错误监控**：无 Sentry 风格的异常捕获和聚合，错误仅通过 stderr/logging 输出
- **无交互式成本报告**：缺少类似 Claude Code `/cost` 命令的会话级成本分解
- **日志非结构化**：Python logging 输出为人类可读文本，不适合程序化分析和告警

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
