---
title: Evaluation & Observability
aliases: [评估, observability, metrics, tracing]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[tool-system]]"
    type: uses
sources: [claude-code]
---

## 一句话定义

Agent 跑得好不好怎么知道 — 效果评估、运行指标、链路追踪。

## 核心问题

- 怎么定义"agent 做得好"？评估指标是什么？
- 运行时能看到什么（token 用量、工具调用次数、延迟）？
- 怎么做事后分析（trace、replay）？
- 评估是离线跑还是在线跑？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | 四层 observability 基础设施：analytics event pipeline（先队列后 sink 解耦）、cost/token tracking（跨 session 连续）、feature gate（GrowthBook 动态门控）、headless profiling；可观测性作为 agent runtime 的一等公民 |
| 关键特点 | 类型系统强制 PII 安全（`never` marker type 在编译期阻止 string 直接进 analytics）；`_PROTO_*` 字段约定区分数据访问级别；user bucket（SHA-256 分 30 bucket）兼顾监控精度与隐私保护 |
| 局限 | Datadog 仅限 firstParty 提供商，第三方部署无 Datadog 数据；cost tracking 在 session 结束才 flush，长任务 budget 超限检测有延迟 |

## 设计权衡

- **编译期 PII 安全 vs 运行期检查**：用 `never` 类型的 marker type 在编译期阻止把 string 直接传入 analytics，把数据安全审查前移而非依赖 code review 或运行时检查——安全成本换取系统级可信度。
- **先队列后 sink vs 直接发送**：sink 未初始化前事件进队列，避免 startup 窗口内丢事件；`queueMicrotask` drain 不阻塞启动关键路径——用轻微的架构复杂度换取事件完整性。

## L2 详情

- [[evaluation-observability--claude-code]]
