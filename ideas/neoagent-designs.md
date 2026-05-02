---
title: neoagent 待抽取的原创设计
domain: neoagent-designs
updated: 2026-04-19
---

# neoagent 待抽取的原创设计

从 `shelf/neoagent/SHELF.md` 列出的"候选提升项"中摘录，等待升级到 `wiki/_insights/` 或 `wiki/_patterns/`。

已升级（不在此列）：
- ~~`auto_free_after` 协议驱动的 freed/recall 循环~~ (deprecated 2026-04-20) → `wiki/_patterns/tool-metadata-driven-context-lifecycle.md` (v2 replaced with MANDATORY_CONTEXT_RULES + WORKING_MEMORY + global compression + LLM-visible free/recall tools)

---

## 2026-04-19 — Session-scoped MCP promoted_tools + ContextVar 并发隔离

**status**: inbox  
**potential_target**: `wiki/_insights/session-scoped-tool-visibility.md`

**核心想法**：MCP 工具默认 deferred（隐藏），`tool_search` 按需提升。**谁被提升**存在 `SessionState.promoted_tools`（per-session），而非进程级单例——配合 ContextVar 传 session，并发 HTTP 请求各自维护独立可见工具集。

**为什么值得**：AgentScope、DeerFlow 的 `promoted_tools` 都是进程级单例，多 session 并发有数据污染隐患；这是 agent 框架做 SaaS 时的**隐性地雷**，同类源没讨论过；解法优雅（单例 deferred registry 真源 + per-session promoted set 视图 + ContextVar 传递）。

**内容骨架**：问题（多 session 并发 MCP 可见性冲突）/ 传统做法（全局 promoted，并发不安全）/ neoagent 做法 / 关键代码证据（`core/loop.py` 的 `_is_hidden()` + `session.py` 的 `promoted_tools`）/ 代价。

**落地**：作为 `wiki/_insights/session-scoped-tool-visibility.md` 发表。20-30 分钟细读 neoagent 源码写清楚。

**相关**：`shelf/neoagent/wiki/_impl/tool-system--neoagent.md`、`wiki/mcp-skills.md`、`wiki/tool-system.md`

---

## 2026-04-19 — Hook + Event 双轨正交分离

**status**: inbox  
**potential_target**: `wiki/_insights/hook-event-dual-track.md`

**核心想法**：把"拦截/改写运行流"和"观测运行流"做成物理解耦的两条路径：
- **Hook** — pre/post 拦截，`HookResult(allow | deny | modify)`，**能影响执行**
- **Event** — `EventBus` 广播 frozen dataclass，多订阅者监听，**只观测不影响**

两个系统零 import 交叉。

**为什么值得**：Claude Code 把 Hook 和 Event 混在一起，一个 handler 既观测又能改——**滥用风险高**（观测者意外改运行流）。物理分开后，权限按路由自然分流：审计/日志订阅 Event 零风险，权限/安全策略挂 Hook 明确可改。400 行代码实现等价于 Claude Code 28+ event 类型的可扩展性。

**内容骨架**：问题（观测+拦截混在一起的坑）/ neoagent 双轨做法 / 代码证据（`hooks.py` + `events.py` + `observe_subscriber.py` 零 import 交叉）/ 对比 Claude Code 混合式 / 迁移建议。

**落地**：通读 neoagent `hooks.py` + `events.py` 两个文件（都 <300 行），写成 `wiki/_insights/hook-event-dual-track.md`。

**相关**：`shelf/neoagent/wiki/_impl/hooks--neoagent.md`、`wiki/hooks.md`、`wiki/evaluation-observability.md`

---

## 2026-04-23 — OTel + Langfuse 作为 neoagent 可观测性基础设施

**status**: inbox  
**potential_target**: `wiki/_patterns/agent-observability-stack.md` 或 `wiki/_insights/otel-langfuse-agent-tracing.md`

**核心想法**：用 OpenTelemetry（标准链路追踪协议）+ Langfuse（LLM 专用可观测平台）组合，为 neoagent 建立完整的可观测层。OTel 负责跨进程 trace 采集和传输，Langfuse 负责 LLM 特有的语义记录（prompt、response、token 用量、cost）。两者互补，不锁定供应商。

**接入设计**：在 neoagent 现有 EventBus/Observer 框架上挂载 `TracingSubscriber`，把 neoagent 的 10 种事件映射为 OTEL Span + Langfuse Generation，无需改动核心循环代码：
- `ProviderRequestEvent` → `llm.generate` span start
- `ProviderResponseEvent` → span end（自动记录 token/cost）
- `ToolCallEvent/ToolResultEvent` → `tool.{name}` 子 span
- `TurnCompleteEvent` → turn-level span boundary
- Session 级 trace 在 `QueryLoop.__call__` 入口创建

**为什么值得**：
1. 可观测性是"全智能体公司"基建的必备层——没有它，无法回答"Agent 花了多少钱、哪一步最慢、为什么失败"
2. neoagent 的 EventBus 设计让接入成本极低（2-3 天），不需要大改核心代码
3. Langfuse 是 2023 年后出现的 LLM 专用工具，传统 APM（Datadog/New Relic）不具备 prompt/cost 原生追踪能力
4. OTel 保证不锁定供应商，未来可无损迁移到其他 backend

**落地**：
1. 写 `neoagent/observe/tracing.py`（TracingSubscriber + LangfuseSubscriber）
2. Langfuse 自托管部署（docker-compose）
3. 在 trip-os 上跑通第一个端到端 trace，验证成本归因 accuracy
4. 记录接入过程踩坑 → 沉淀为知识库 L2 页

**风险**：
- Langfuse 成熟度不如传统 APM，社区较小，长期维护风险
- 跨进程 trace 传播（MCP Server、subagent）需要显式传递 `traceparent`，容易遗漏
- 异步场景下 Python contextvars 可能丢失，需显式 `copy_context()`
- Agent 行为回放（保存完整 context 可重跑）Langfuse 不支持，需自建

**相关**：`wiki/evaluation-observability.md`、`wiki/tool-system.md`、`wiki/query-loop.md`、`shelf/neoagent/wiki/_impl/hooks--neoagent.md`
