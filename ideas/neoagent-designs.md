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
