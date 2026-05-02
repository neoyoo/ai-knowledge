---
title: neoagent 接入 Langfuse —— OTEL 标准 + 自托管，分阶段实施
date: 2026-04-22
status: accepted (planned, not yet implemented)
decider: neo
related-source: raw/langfuse/ (待 symlink)
related-wiki: [[observability]], [[tool-system]], [[multi-agent]]
---

## 背景

neoagent v3.2d 已完成（866 tests，11 维度覆盖）。当前可观测性靠内部 EventBus + ObserverSubscriber（本地 JSONL 日志），进程内有完整事件流，但缺少：

1. **UI 可视化** —— 看不到 trace 树形结构
2. **跨进程链路** —— MCP subprocess / 未来多节点无法串联
3. **跨 session 聚合查询** —— 本地日志散落，无法查"过去 7 天 gpt-4 调用 p95"
4. **Prompt 版本管理** —— 目前零支持
5. **Cost 归因** —— Token 级成本没有聚合

Neo 的核心诉求：**想像 SkyWalking 一样看到 LLM 实际交互的分布式链路**——主 agent 调用了什么 skill、什么 MCP、spawn 了什么 subagent、subagent 又干了什么。

## 候选方案对比

| 方案 | 优点 | 缺点 | 推荐 |
|------|------|------|------|
| **Langfuse 官方 SDK**（绑死 Langfuse） | 集成快、Prompt/Dataset/Eval 一体 | 锁定 Langfuse；换 backend 要重写 | ⚠️ 备选 |
| **OTEL 标准 + Langfuse 作为 backend** | 未来可换 Jaeger/SkyWalking；生态成熟 | 多 1-2 天适配成本 | ✅ |
| **Arize Phoenix**（单 SQLite） | 轻量、本地零运维 | 无 Prompt 管理 / Cost / Eval 闭环 | ❌ |
| **自研 mini UI**（FastAPI + neoagent Observer 日志） | 零外部依赖 | 重复造轮子；Prompt/Eval/Dataset 全自己写 | ❌ |

## 决策

**采用 OTEL 标准（W3C traceparent）+ Langfuse 作为默认 backend，自托管部署。**

### 选 OTEL 不选 Langfuse SDK 的理由

- 数据格式标准化，backend 可替换（Jaeger、SkyWalking、Tempo、Datadog 都兼容）
- Langfuse v3+ 已原生接受 OTEL 数据（通过 OTLP endpoint）
- contextvars + OTEL SDK 是 Python 标准做法，生态工具多

### 选自托管不选 Cloud 的理由

- Langfuse 核心 MIT 许可，自托管零限制（仅 SSO/多租户/审计日志/Billing 是 EE 闭源，对我们无影响）
- 数据主权、无按量计费
- Docker Compose 一键起（postgres + clickhouse + redis + minio + web + worker）

## 技术架构

### Trace Context 传递模型

```
全局 TraceID（一次用户请求贯穿）
 └─ Orchestrator root span
     ├─ GENERATION (LLM call)
     ├─ TOOL (dispatch_worker)
     │   └─ Worker sub-span (ParentSpanID 对齐)
     │       ├─ GENERATION
     │       ├─ TOOL (MCP call)  ← _meta.traceparent 注入
     │       │   └─ MCP server 内部 span
     │       └─ GENERATION
     └─ GENERATION (final)
```

### 各边界传递方式

| 边界 | 方案 | 实施难度 |
|------|------|----------|
| agent → skill / tool（进程内） | `contextvars` 维护 span stack | 容易 |
| agent → sub-agent（v3.2b Task Envelope） | Envelope 加 `trace_context` 字段 | 容易 |
| HTTP channel 入口（v3.2c） | 读 W3C `traceparent` header → root span | 容易 |
| **agent → MCP server（stdio 子进程）** | MCP 请求 `_meta.traceparent` 字段 | 中等 |
| sub-agent → 其内部 tool/MCP | 同主 agent 逻辑，嵌套继承 | 免费 |

### EventBus → OTEL Span 映射

| neoagent 事件 | → OTEL Span | 备注 |
|---|---|---|
| `ProviderRequestEvent` + `ProviderResponseEvent` | GENERATION span | model/tokens/cost 进 attributes |
| `ToolCallEvent` + `ToolResultEvent` | TOOL span | 参数/返回值进 input/output |
| `CompressCheckEvent` + `CompressDoneEvent` | 自定义 SPAN（name="context_compression"） | neoagent 独有概念 |
| `HookEvent` | Event（非 span） | 挂在当前 span 上 |
| Orchestrator → Worker spawn | 新 sub-span + parent 链接 | v3.2b 集成 |
| Session | Trace `sessionId` attribute | 多轮对话串联 |

## 分阶段实施

**Phase 1 — 单进程链路打通（1 周）**
- 新增 `neoagent/observability/tracing.py`：`TraceContext`（contextvars）+ span lifecycle
- 新增 `neoagent/integrations/langfuse/` extras 模块：`LangfuseObserver(Observer)`
- 走 OTEL SDK（`opentelemetry-api` + `opentelemetry-sdk` + `opentelemetry-exporter-otlp`）
- EventBus 订阅：映射 Provider/Tool/Compress/Hook 四类事件
- Task Envelope 加 `trace_context` 字段（v3.2b worker spawn 时自动传）
- 验收：orchestrator → worker → tool → LLM 全链路在 Langfuse UI 上呈现嵌套树

**Phase 2 — MCP 边界穿透（1 周）**
- neoagent MCP client 注入 `_meta.traceparent`
- 自研 MCP server shim（拦截 request → 建 span → 回写 response）
- 第三方 MCP server 若不配合，就只显示入口 TOOL span（可接受）
- 验收：常见 MCP 调用（brave_search、filesystem 等）在 UI 上可见内部 span

**Phase 3 — 选做**
- HTTP channel（v3.2c）入口读 `traceparent` header
- 外部 HTTP 调用（如果 agent 主动发 HTTP）注入 `traceparent`
- Prompt 版本管理接入（Langfuse Prompt API）
- Cost 归因（按 agent / skill / user 维度）

## 发布策略

- 作为 neoagent **optional extras**：`pip install neoagent[langfuse]` 或 `pip install neoagent[tracing]`
- 默认 off，不影响 core 依赖
- 文档：`docs/integrations/langfuse.md` + 示例 compose 文件

## 后果与约束

### 正面
- 彻底解决多 agent 协作调试痛点
- 沉淀 OTEL 集成能力，将来换 backend 零重写
- Prompt/Eval/Cost/Dataset 高级功能免费获得

### 需要评估
- **Langfuse 自托管运维成本**：4 个后端服务（postgres + clickhouse + redis + minio）+ web/worker，内存基线 2-3 GB。个人/小团队可接受，生产部署需监控
- **Telemetry 默认开启**：`TELEMETRY_ENABLED=true` 回传匿名使用数据到 Langfuse GmbH，介意需显式关闭
- **ENCRYPTION_KEY / SALT 必须改**：默认值是占位符，生产用必换

### 开发摩擦
- OTEL Python SDK 异步语义有坑（contextvars 在 asyncio.create_task 后的继承），需测试充分
- MCP server 是第三方代码，`_meta` propagation 不一定都配合

## 未来触发点

- **立刻做**：等待 v3.2d 合并 main 后启动 Phase 1
- **v3.3 正式功能**：Phase 1 + Phase 2 作为 v3.3 里程碑之一
- **规模化**：若 trace 量超出单机 ClickHouse 能力（百万事件/天），考虑 Langfuse Cloud 或自建 ClickHouse 集群

## 相关资料

- Langfuse 源码：`raw/langfuse/`（待 symlink）
- Langfuse 核心 License：MIT（`ee/` 目录外全开源）
- EE 闭源功能（对我们无影响）：admin-api / audit-log-viewer / billing / multi-tenant-sso / sso-settings / ui-customization
- OTEL Python：https://opentelemetry.io/docs/languages/python/
- W3C Trace Context：https://www.w3.org/TR/trace-context/
