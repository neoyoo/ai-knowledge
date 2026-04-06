---
title: "Evaluation Observability — Claude Code"
category: L2
parent: "[[evaluation-observability]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 把可观测性作为 agent runtime 的一等基础设施，而非事后补洞。整套 observability 系统由四层构成：analytics event pipeline（事件日志）、cost/token tracking（费用追踪）、feature gate（动态特性门控）、以及 debug profiling（性能 profiling）。这套系统同时支持交互模式和 headless SDK 模式，是 Claude Code 能在生产环境稳定运行的关键工程支撑。

## 架构分析

### Analytics Event Pipeline

analytics 系统采用"先队列后 sink"的解耦架构：

**入口层**（`src/services/analytics/index.ts`）：
- `logEvent()` / `logEventAsync()` 是统一 API，无依赖，不会形成 import cycle
- sink 未初始化前，事件进入 `eventQueue[]`，避免 startup 时间窗口内丢事件
- `attachAnalyticsSink()` 初始化时异步用 `queueMicrotask()` drain 队列，不阻塞启动路径

**Sink 层**（`src/services/analytics/sink.ts`）：
- 双后端路由：Datadog（外部监控）+ 1P event logging（内部 BigQuery）
- `shouldSampleEvent()` 做事件采样，由 `tengu_event_sampling_config` 动态配置控制
- PII 保护：`_PROTO_*` key 命名约定——这类字段只进 1P 特权列，Datadog fanout 前由 `stripProtoFields()` 剥离

**Datadog 层**（`src/services/analytics/datadog.ts`）：
- 批量发送：`logBatch[]` 积累，15 秒或达到 100 条时 flush
- allowlist 控制（`DATADOG_ALLOWED_EVENTS`）：并非所有内部事件都上报 Datadog，精确控制上报范围
- cardinality 优化：MCP tool name 统一归类为 `"mcp"`，外部用户的 model 名归一化为短名，dev 版本号截断去 sha
- user bucket：用 SHA-256 hash 把 user ID 分入 30 个 bucket，既能估算受影响用户数，又保护隐私

**数据安全约束**：
```typescript
// 类型系统强制标注，防止误记 code/filepath
type AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS = never
```
logEvent 的 metadata 类型不接受普通 string，必须显式标注"已验证不含代码/路径"，把安全审查前移到编译期。

### 主要 Event 类型

Datadog allowlist 揭示了 Claude Code 关注的核心 observability 指标：

| 事件类 | 含义 |
|------|------|
| `tengu_init` / `tengu_started` / `tengu_exit` | session 生命周期 |
| `tengu_api_success` / `tengu_api_error` / `tengu_query_error` | LLM API 调用质量 |
| `tengu_tool_use_success` / `tengu_tool_use_error` | 工具执行结果 |
| `tengu_tool_use_granted_*` / `tengu_tool_use_rejected_*` | 权限审批追踪 |
| `tengu_compact_failed` | context 压缩失败 |
| `tengu_cancel` | 用户取消操作 |
| `tengu_oauth_*` | 认证流程追踪 |
| `tengu_model_fallback_triggered` | 模型降级事件 |
| `tengu_uncaught_exception` / `tengu_unhandled_rejection` | 全局错误捕获 |
| `tengu_team_mem_sync_*` | 团队记忆同步 |

### Feature Gate 系统（GrowthBook）

`src/services/analytics/growthbook.ts` 集成 GrowthBook 做动态特性门控：

- 用户属性维度包括：`sessionId`、`deviceID`、`platform`、`organizationUUID`、`subscriptionType`、`rateLimitTier`、`appVersion`
- 支持 GitHub Actions 环境识别（`GitHubActionsMetadata`），针对 CI 场景做特化控制
- A/B 实验结果自动上报 1P event logging（`logGrowthBookExperimentTo1P()`）
- feature gate 状态有本地 cache，初始化前 fallback 到 cache 值（`checkStatsigFeatureGate_CACHED_MAY_BE_STALE`），避免启动时窗口内丢失控制

代码中的 `feature('XXX')` 调用（如 `feature('CONTEXT_COLLAPSE')`、`feature('COORDINATOR_MODE')`）在 bundle 时做 dead code elimination，feature flag 关闭的分支不进最终产物。

### Cost & Token Tracking

`src/cost-tracker.ts` 是 claude code 最重要的可观测指标之一，跨 session 连续追踪：

追踪维度：
- `totalCostUSD` — 累计费用（USD）
- `totalAPIDuration` / `totalAPIDurationWithoutRetries` — API 时延（含/不含重试）
- `totalToolDuration` — 工具执行总时间
- `totalInputTokens` / `totalOutputTokens` — token 消耗
- `totalCacheReadInputTokens` / `totalCacheCreationInputTokens` — prompt cache 命中详情
- `totalWebSearchRequests` — web search 调用次数
- `totalLinesAdded` / `totalLinesRemoved` — 代码行数变更统计
- `modelUsage` — 按 model 分桶的用量（支持多模型混用场景）

`QueryEngine` 在每次 query 循环内累积 usage：通过 `accumulateUsage()` 更新 session-level 累计值，在 session 结束时通过 `flushSessionStorage()` 持久化，resume 时通过 `restoreCostStateForSession()` 恢复。

`maxBudgetUsd` / `taskBudget` 参数在 `QueryEngine` config 中定义，cost tracker 的实时值驱动 budget 控制逻辑。

### Permission Denial Tracking

`QueryEngine` 专门维护 `permissionDenials: SDKPermissionDenial[]`，记录每次权限被拒绝的工具调用。这些数据：
- 在 SDK 模式下作为结构化输出暴露给调用方
- 用于 `tengu_tool_use_rejected_*` 事件上报

设计意图是让 SDK 用户能区分"模型出错"和"权限被拒"，精确诊断 agent 行为。

### Headless Profiling

`src/utils/headlessProfiler.ts` 的 `headlessProfilerCheckpoint()` 在 `QueryEngine.submitMessage()` 中被调用，记录 headless 模式下各阶段的时间戳。这是 SDK 性能 benchmark 的基础设施，不影响交互模式。

### 关键代码路径
- `src/services/analytics/index.ts` — event 队列与 sink 接口，无依赖入口
- `src/services/analytics/sink.ts` — 双后端路由（Datadog + 1P）
- `src/services/analytics/datadog.ts` — Datadog batch sender，cardinality 优化
- `src/services/analytics/growthbook.ts` — feature gate，A/B 实验
- `src/cost-tracker.ts` — cost/token/duration 追踪，跨 session 持久化
- `src/QueryEngine.ts` — `permissionDenials` 追踪，usage 累积
- `src/utils/headlessProfiler.ts` — headless 模式性能 checkpoint

## 设计亮点

- **类型系统强制 PII 安全**：用 `never` 类型的 marker type 在编译期阻止把 string 直接传入 analytics，把数据安全审查前移而非依赖 code review
- **Sink 解耦 + 队列缓冲**：analytics 模块无依赖，startup 前事件不丢失；`queueMicrotask` drain 不阻塞启动关键路径
- **`_PROTO_*` 字段约定**：通过命名约定区分不同访问级别的数据，一个 `stripProtoFields()` 保护所有非特权后端，不需要 per-sink 配置
- **user bucket 设计**：30 个固定 bucket 既能估算 unique users（用于 alerting），又不在 Datadog 存储明文 user ID，保护隐私同时支持运营监控
- **cost tracker 跨 session 连续性**：budget 控制数据不因进程重启而重置，长任务的费用控制可靠

## 局限性

- **Datadog 仅限 firstParty 提供商**：Bedrock/Vertex/Foundry 用户不会产生 Datadog 数据，第三方部署的可观测性需要自行搭建
- **analytics 只追踪布尔/数字，不追踪字符串**：类型约束阻止了直接记录错误 message、工具名称等文本信息，需要通过 `AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS` 显式豁免
- **feature gate 依赖网络初始化**：GrowthBook 需要从服务端拉取 feature 定义，冷启动时 fallback 到本地 cache，可能短暂产生与服务端不一致的行为
- **cost tracking 无 real-time streaming**：`totalCostUSD` 在 session 结束才 flush，长任务运行中 budget 超限的检测存在延迟

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
