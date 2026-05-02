---
title: OpenTelemetry Eval Bridge
aliases: [OTel eval bridge, tracing-to-eval, observability evaluation bridge]
kind: pattern
created: 2026-05-02
updated: 2026-05-02
concepts_involved: [[evaluation-observability]], [[finetuning-system]]
reference_implementations: [agentscope]
status: mature
---

## 一句话定义

用同一套 OpenTelemetry tracing 同时服务生产观测和离线评测，让 eval runner 通过内存 exporter / baggage 自动收集 token、工具调用、延迟和轨迹指标。

## 触发问题

评测系统常常重复实现一套指标采集：workflow 里手写 token 统计、工具调用计数、耗时记录。生产观测又有另一套 tracing。两套系统字段不一致，评测结果和线上行为难以对齐。

## 参与的概念

- [[evaluation-observability]] — tracing、span、metrics、eval storage
- [[finetuning-system]] — model selection / prompt optimization / judge 需要真实成本和过程指标

## 核心设计

### 1. 运行时调用统一打 span

Agent、LLM、Tool、Embedding、Formatter 等调用层用装饰器产生 OTel span。span 属性复用 GenAI semantic conventions，并追加框架自有扩展属性。

### 2. Eval runner 切换 exporter

生产环境把 span 发到 Jaeger / Grafana / Langfuse。评测环境切换到 `_InMemoryExporter`，不改 workflow 代码即可把 span 留在本进程。

### 3. Baggage 绑定 task / repeat

评测时用 OTel baggage 标记 `task_id`、`repeat_id`。Exporter 根据 baggage 把 span 路由到对应样本，自动聚合 token、tool calls、latency。

### 4. Judge 读取 metrics

模型选择、prompt 优化或 finetuning judge 不只看最终答案，也能按 token 成本、耗时、工具调用次数或 trajectory 质量评分。

## 适用场景

- agent benchmark / regression eval
- 多模型候选选择
- prompt optimization / DSPy 类自动搜索
- finetuning 数据生成与过滤
- 需要对齐线上 tracing 与离线评测指标

## 不适用场景

- 一次性脚本评测，指标只需人工查看
- tracing 系统不能记录任何 prompt/response 元数据的高敏场景
- 多线程 / 多进程上下文传播不可控且没有 baggage 兜底方案

## 参考实现

AgentScope tracing + evaluate：

- `wiki/_impl/evaluation-observability--agentscope.md` — `_InMemoryExporter` + OTel Baggage 桥接 eval 指标
- `wiki/_impl/finetuning-system--agentscope.md` — `select_model` 用 tracing token usage 注入 judge metrics

## 迁移 checklist

- 先定义 span 属性白名单，避免把敏感 prompt 全量写进监控
- 为 eval runner 提供内存 exporter 模式
- 用 baggage 或等价上下文机制绑定 task id
- 指标聚合和 judge 输入使用同一 schema
- 流式响应要等最后一个 chunk 后再关闭 span

## 常见陷阱

- tracing 记录完整用户数据，评测/监控系统变成隐私风险点
- baggage 在多线程环境丢失，span 静默无法归属到样本
- finish reason 写死，无法区分 stop / length / tool_use
- eval 指标和生产指标字段名不同，后续无法对齐

## 相关概念

- [[evaluation-observability]]
- [[finetuning-system]]

