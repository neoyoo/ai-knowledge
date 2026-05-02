---
title: Evaluation & Observability
aliases: [评估, observability, metrics, tracing]
category: L1
created: 2026-04-06
updated: 2026-04-25
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[tool-system]]"
    type: uses
sources: [claude-code, openharness, deer-flow, hermes-agent, agentscope]
---

## 一句话定义

Agent 跑得好不好怎么知道 — 效果评估、运行指标、链路追踪。

## 核心问题

- 怎么定义"agent 做得好"？评估指标是什么？
- 运行时能看到什么（token 用量、工具调用次数、延迟）？
- 怎么做事后分析（trace、replay）？
- 评估是离线跑还是在线跑？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent | AgentScope |
|------|------------|-------------|----------|--------------|-----------|
| 核心设计 | 四层 observability 基础设施：analytics event pipeline（先队列后 sink 解耦）、cost/token tracking（跨 session 连续）、feature gate（GrowthBook 动态门控）、headless profiling；可观测性作为 agent runtime 的一等公民 | 围绕 token 计数和结构化事件流两个支柱：`CostTracker` 跨 turn 累计 usage 统计；`--output-format stream-json` 以换行分隔 JSON 事件流（assistant_delta/tool_started/tool_completed/assistant_complete）暴露实时执行过程 | 以 LangGraph 原生 SSE 流式输出为核心，提供模型 token、工具调用、subagent 任务状态的实时可见性；两个隐式评估机制：memory 置信度分数作为轻量级质量信号，`LoopDetectionMiddleware` 作为运行时自评估守卫 | 双轨设计：运行时可观测性（SessionDB SQLite WAL + 多平台写入 + InsightsEngine 多维分析）和 RL 训练评估（BatchRunner 轨迹生成 + TrajectoryCompressor 压缩 + WandB 指标 + 推理测试）两条线相对独立；本地优先，所有数据写入 `~/.hermes/` OTel 标准 + 内置评估管道的双模块设计：`tracing/` 通过五类专用装饰器（`trace_reply`/`trace_llm`/`trace_toolkit`/`trace_embedding`/`trace_format`）为 Agent/LLM/Tool/Embedding/Formatter 各调用层注入 span，全部遵循 OpenTelemetry GenAI Semantic Conventions；`evaluate/` 提供 Benchmark-Task-Metric-Evaluator-Storage 五层评估管道，通过 `_InMemoryExporter` + OTel Baggage 把运行时追踪数据自动桥接为评测指标统计，实现"评测时自动采集 token/工具调用次数"的闭环；支持 GeneralEvaluator（串行）和 RayEvaluator（Ray 并发）两种评测执行模式 |
| 关键特点 | 类型系统强制 PII 安全（`never` marker type 在编译期阻止 string 直接进 analytics）；`_PROTO_*` 字段约定区分数据访问级别；user bucket（SHA-256 分 30 bucket）兼顾监控精度与隐私保护 | `stream-json` 模式提供结构清晰的事件流，外部工具可直接 pipe 消费（如 `jq`、CI 脚本）；tool_started/tool_completed 配对结构便于计算工具调用延迟；零依赖，不引入 OpenTelemetry 等重型依赖 | Loop detection 即运行时自评估（无需外部框架直接在执行路径中拦截质量问题）；memory confidence 作为隐式反馈信号（将用户纠错行为转化为质量数据）；SSE 细粒度流式可见性（subagent 任务级别状态广播）；零配置上手 | `normalize_usage()` 统一 Anthropic/OpenAI/Codex 三种 API 格式；`pricing_version` per-session 版本字符串（如 `anthropic-prompt-caching-2026-03-16`）支持费用审计溯源；BatchRunner 三道数据质量门禁（零推理过滤、工具名幻觉过滤、schema 归一化）；`LOCKED_FIELDS` 保障 RL 实验可复现性 OTel 标准属性 + `agentscope.*` 扩展双层命名，trace 数据无需适配层即可接入 Jaeger/Grafana/Langfuse；`_InMemoryExporter` 既可连外部 OTLP 后端（生产观测），又可在评测时切换为内存导出（指标统计），同一套 tracing 基础设施两用；`FileEvaluatorStorage` 在每次执行前检查已有结果文件，天然支持断点续评；ACEBenchmark 内置中文手机 Agent 工具调用基准（ACEAccuracy 最终状态 + ACEProcessAccuracy 里程碑路径），支持过程准确率（trajectory 级别）而非只看最终结果；流式 LLM 响应通过 generator wrapper 在最后一个 chunk 后才写 span 属性，保持 span 完整性 |
| 局限 | Datadog 仅限 firstParty 提供商，第三方部署无 Datadog 数据；cost tracking 在 session 结束才 flush，长任务 budget 超限检测有延迟 | 仅 token 数量，无美元成本估算；未集成 OpenTelemetry，无 trace/span 和分布式追踪能力；无错误监控，无 Sentry 风格的异常捕获；日志为 Python logging 非结构化输出 | 无结构化遥测（不支持 OpenTelemetry）；无成本追踪；无自动化 eval 框架；仅实时可观测（SSE 事件不持久化，无法历史 trace 回溯） | 无外部 telemetry sink（无 Datadog/Sentry/OpenTelemetry），大规模部署 log 分析全靠 grep；`rl_check_status()` 30 分钟全局 rate limit，训练初期无法获得状态反馈；WandB 强依赖，不可用时指标全部为空；`_active_runs` 进程内存状态，重启后训练状态不可恢复 `finish_reason` 硬编码为 `"stop"`（有 FIXME 注释），不反映实际 `tool_use`/`length` 等终止原因；`_InMemoryExporter` 依赖 Baggage 上下文，多线程环境下传播失败时统计数据静默丢弃；GeneralEvaluator 串行执行 Metric，多指标场景效率低（需 Ray 依赖才能并行）；ACEBenchmark 仅支持中文数据集，英文版本注释未启用；tracing 装饰器主要覆盖 async 路径，同步自定义模型需手动添加 |

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

**多提供商 cost tracking 的格式陷阱**：Anthropic、OpenAI、Codex 返回的 token usage 字段名称和语义不统一（如 OpenAI 的 `cached_tokens` 是从总量里扣出来的，而 Anthropic 的 `cache_read_input_tokens` 是独立字段）。如果直接把不同提供商的返回值做加法，计算结果会出错。Hermes Agent 的 `normalize_usage()` 通过 `CanonicalUsage` 统一格式，是跨提供商 cost tracking 的必要前置步骤。

**RL 训练数据没有质量门禁，垃圾进垃圾出**：直接把 agent 跑出来的 trajectory 送去训练，没有过滤推理缺失（模型只输出答案没有思考过程）、工具名幻觉（模型调用了不存在的工具）等低质样本，会让训练数据污染模型。Hermes BatchRunner 的三道过滤（零推理过滤、工具名幻觉过滤、schema 归一化）针对的就是这三类常见污染源。

**RL 实验基础参数被随意修改，结果无法复现**：在 agent 控制 RL 训练参数时，若 tokenizer 路径、LoRA rank、学习率等基础参数暴露给模型修改，模型可能为了"让训练跑起来"而乱调参数，导致不同 run 之间条件不一致，实验结果无法对比。Hermes 的 `LOCKED_FIELDS` 机制将基础参数硬编码并拒绝 agent 修改，只暴露业务参数（数据集、步数、名称），是 RL 工具链中值得采用的保护模式。

**本地优先 vs 可扩展性的取舍**：Hermes 的"本地优先"（所有数据落 `~/.hermes/` SQLite，无外部 sink）在单机开发场景极易上手，但多节点部署时无法汇聚数据，也没有实时告警能力。这是一种有意识的取舍，不是设计缺陷——关键是在项目初期明确自己处于哪个阶段，避免用生产级 telemetry 基础设施解决一台机器上的调试问题，也避免到了多节点时才发现数据全在本地无法汇总。

## L2 详情

- [[evaluation-observability--claude-code]]
- [[evaluation-observability--openharness]]
- [[evaluation-observability--deer-flow]]
- [[evaluation-observability--hermes-agent]]
- [[evaluation-observability--agentscope]]
