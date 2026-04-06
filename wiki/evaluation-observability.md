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
sources: [claude-code, openharness]
---

## 一句话定义

Agent 跑得好不好怎么知道 — 效果评估、运行指标、链路追踪。

## 核心问题

- 怎么定义"agent 做得好"？评估指标是什么？
- 运行时能看到什么（token 用量、工具调用次数、延迟）？
- 怎么做事后分析（trace、replay）？
- 评估是离线跑还是在线跑？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | 四层 observability 基础设施：analytics event pipeline（先队列后 sink 解耦）、cost/token tracking（跨 session 连续）、feature gate（GrowthBook 动态门控）、headless profiling；可观测性作为 agent runtime 的一等公民 | 围绕 token 计数和结构化事件流两个支柱：`CostTracker` 跨 turn 累计 usage 统计；`--output-format stream-json` 以换行分隔 JSON 事件流（assistant_delta/tool_started/tool_completed/assistant_complete）暴露实时执行过程 |
| 关键特点 | 类型系统强制 PII 安全（`never` marker type 在编译期阻止 string 直接进 analytics）；`_PROTO_*` 字段约定区分数据访问级别；user bucket（SHA-256 分 30 bucket）兼顾监控精度与隐私保护 | `stream-json` 模式提供结构清晰的事件流，外部工具可直接 pipe 消费（如 `jq`、CI 脚本）；tool_started/tool_completed 配对结构便于计算工具调用延迟；零依赖，不引入 OpenTelemetry 等重型依赖 |
| 局限 | Datadog 仅限 firstParty 提供商，第三方部署无 Datadog 数据；cost tracking 在 session 结束才 flush，长任务 budget 超限检测有延迟 | 仅 token 数量，无美元成本估算；未集成 OpenTelemetry，无 trace/span 和分布式追踪能力；无错误监控，无 Sentry 风格的异常捕获；日志为 Python logging 非结构化输出 |

## 设计权衡

### 方案对比

| 方案 | 核心机制 | 实施成本 | 诊断能力 | 适用规模 |
|------|---------|---------|---------|---------|
| 无观测（Blind Mode） | 只看最终输出 | 零 | 无（出错只能猜） | 原型/个人实验 |
| 日志 + Token 计数 | 基础日志 + input/output token 累计（OpenHarness CostTracker 方案） | 极低 | 低（能看用量，无法追问题） | 开发阶段 |
| 结构化追踪（Tracing） | OpenTelemetry spans，工具级延迟，每轮 token 分解 | 中（需 collector + dashboard） | 高（精确归因延迟和成本） | 生产部署 |
| 自动评估（Eval Framework） | 自动化测试套件，评估正确性/安全/效率 | 高（eval 设计 + 维护） | 系统性（回归防护，模型对比） | 产品化/研究 |

### 场景决策指南

**个人项目 / 实验阶段** → 日志 + Token 计数即够。最低投入获得基本可见性：知道每次调用花了多少 token，出错时有日志可查。OpenHarness 的 `stream-json` 模式（`--output-format stream-json`）是好起点，可以直接 `jq` 消费。

**小团队产品上线** → 至少要上结构化追踪，重点是 cost tracking。没有按请求的费用归因，月底账单会是惊喜。最低可行方案：每个请求记录 model/input_tokens/output_tokens/latency，存到任何数据库。不需要完整 OpenTelemetry 栈。

**企业级 / 多用户部署** → 追踪 + 自动评估双轨并行。追踪负责运营健康（延迟 P99、成本、错误率），Eval 负责质量保证（模型升级不退步、新功能不破坏旧行为）。Claude Code 的四层 observability 基础设施（analytics pipeline → cost tracking → feature gate → headless profiling）代表了这个量级的参考实现。

**研究 / 模型比较** → 重点投入 Eval Framework。追踪层对比模型意义不大，需要的是针对具体任务的评估基准：相同 prompt 下 A 模型 vs B 模型的正确率、拒绝率、成本效率。

### 常见陷阱

**没有 cost tracking，等到账单才知道**：API 费用在高并发下累积极快。token 计数是最便宜的观测投资，不做就是在裸奔。OpenHarness 的 `CostTracker` 实现仅需跟踪 `input_tokens`/`output_tokens` 累计，几十行代码，没有理由跳过。

**Eval 只测 happy path**：测试套件覆盖了正常对话、工具调用成功，但没有测工具报错、上下文截断、空输入、并发等边缘场景。生产中出问题的恰恰是这些。解法：主动收集生产中的失败案例，反向补入 eval 套件。

**追踪数据未脱敏，用户对话进入监控系统**：结构化追踪天然会记录 span 属性，如果把完整 prompt/response 塞进 span，用户数据就流入了 Datadog/Jaeger 等监控基础设施。Claude Code 用 `never` marker type 在编译期阻止 string 直接进 analytics，是值得借鉴的模式。最低限度：追踪只记录长度和类型，不记录内容。

**长任务中 budget 超限检测滞后**：Claude Code 的 cost tracking 在 session 结束才 flush，这意味着长任务可能超预算后才被发现。如果有 `maxBudgetUsd` 类约束，需要在每轮结束后做实时检查，而不是依赖 session flush。

## L2 详情

- [[evaluation-observability--claude-code]]
- [[evaluation-observability--openharness]]
