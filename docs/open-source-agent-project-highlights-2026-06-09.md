---
title: "Open Source Agent Project Highlights"
date: 2026-06-09
type: source-review
scope: ai-knowledge / agent-os
status: draft
---

# 开源 Agent 项目核心亮点梳理

本文件记录 2026-06-09 本地源码同步后的源码级判断，用作后续维护 `ai-knowledge` wiki 和规划 `agent-os` 的输入。`claude-code-sourcemap` 是用户本地下载的 Claude 源码，本轮明确排除，不纳入同步和分析。

## 版本状态

| 项目 | 本地路径 | 当前版本 / 描述 | 本轮结论 |
|---|---|---|---|
| AgentScope / AgentScope2 | `/Users/neo/Desktop/project/git/agentscope` | `v2.0.1-11-g0e5418e8` | AgentScope2 是同一项目的 2.x 线，不是独立项目。 |
| AgentScope Java | `/Users/neo/Desktop/project/git/agentscope-java` | `2.0.0-SNAPSHOT` | 独立 Java/Spring/Nacos/A2A 项目，不是 Python 仓库子版本。 |
| DeerFlow | `/Users/neo/Desktop/project/git/deer-flow` | `v2.0-m1-rc2-11-g16391e35` | 已是最新；新增 RunEventStore/rollback/tool-output/channel 能力已入 wiki。 |
| Hermes Agent | `/Users/neo/Desktop/project/git/hermes-agent` | `0.16.0` | 已是最新。 |
| OpenHarness | `/Users/neo/Desktop/project/git/OpenHarness` | `v0.1.9-51-g9b2efd7` | 已是最新。 |
| MiroFish | `/Users/neo/Desktop/project/git/MiroFish` | `v0.1.2-46-g96096ea` | 已是最新。 |
| MemPalace | `/Users/neo/Desktop/project/git/mempalace` | `v3.4.0` | 已是最新，旧 KB 摘要需要重写。 |
| SimpleMem | `/Users/neo/Desktop/project/git/SimpleMem` | `v0.3.0-2-g74174a1` | 已是最新，shelf 归档理由需要更新。 |
| TencentDB-Agent-Memory | `/Users/neo/Desktop/project/git/TencentDB-Agent-Memory` | `v0.3.6-8-gf92b102` | 新增 clone，值得纳入 KB 评估。 |

## 一页总览

| 项目 | 最核心亮点 | 最值得进入 KB 的主题 | 对 agent-os 的借鉴优先级 |
|---|---|---|---|
| AgentScope | Python 2.x 面向大型 Web/distributed agent runtime：session / workspace / permission / message bus / team tools / custom agent class。 | `multi-agent`、`sandbox-isolation`、`channel-remote` | P1 |
| AgentScope Java | AgentCard / A2A / Nacos / Spring Boot starter 的企业服务化注册发现链路。 | `agent-registry-discovery`、`multi-agent` | P1 |
| DeerFlow | 生产级运行时可靠性：RunManager、run event store、checkpointer、tool output externalization、replay E2E。 | `session-recovery`、`evaluation-observability`、`tool-system` | P1 |
| Hermes Agent | 平台化 agent shell：provider fallback、gateway、本地/远程文件引用、memory provider fencing、模型目录缓存。 | `channel-remote`、`prompt-system`、`memory-system` | P2 |
| OpenHarness | SDK runtime 与个人 agent 产品层分离，结构化 memory schema + auto-dream consolidation。 | `memory-system`、`hooks`、`channel-remote` | P1 |
| MiroFish | 数字孪生 / 群体仿真的 agent 应用范式：Zep GraphRAG + OASIS + report agent。 | `multi-agent` 应用型案例 | P3 |
| MemPalace | 从单一 raw memory 进化为 backend/source adapter 插件契约 + conformance suite。 | `memory-system`、`mcp-skills`、`evaluation-observability` | P1 |
| SimpleMem | 统一 AutoMemory router + text/omni 后端 + EvolveMem 离线检索策略优化。 | `memory-system`、`evaluation-observability` | P1 |
| TencentDB-Agent-Memory | 短期符号化 offload + 长期 L0/L1/L2/L3 分层 memory，证据链可 drill-down。 | `context-management`、`memory-system`、`tool-system` | P0 |

## AgentScope / AgentScope2

**核心亮点：大型 Web/distributed agent runtime 的服务化骨架。**

AgentScope 2.x 的价值不只是 middleware 或 tool API，而是它把 agent 当作可服务化的运行单元来设计。`create_app()` 以 FastAPI app factory 暴露 storage、message bus、workspace manager、extra agent middleware/tools、custom subagent templates、custom agent class 等扩展点；session、workspace、permission context 共同构成多租户运行边界；team worker 通过 `AgentCreate` 创建并继承 leader workspace/model。

本轮新增 commit `0e5418e8` 把 workspace root 写入 permission context，并把 Docker workspace 的 `host_workdir` 与容器内 `workdir` 区分开。这说明 AgentScope 正在强化“workspace 是安全边界”的设计，而不是只把 workspace 当文件目录。

需要注意：当前 Python 2.x 源码未包含旧版 `src/agentscope/a2a/`、`pipeline/`、`realtime/`、`tts/` 路径；AgentCard / A2A / Nacos 的源码-backed 主参考应转到独立的 AgentScope Java 项目。

**适用范围：**

适合多用户、多 session、多 agent、远程 workspace、Web session runtime 的 agent 平台。对于小型单机 agent，它的抽象会偏重。

**KB 维护建议：**

更新 `wiki/_impl/multi-agent--agentscope.md`、`wiki/_impl/channel-remote--agentscope.md`、`wiki/_impl/sandbox-isolation--agentscope.md`。L1 `agent-registry-discovery` 不应再把 AgentCard/A2A/Nacos 归到 Python AgentScope。

**agent-os 借鉴：**

agent-os 已有 registry、A2A、session affinity、TaskStore 和 ASGI channel。下一步不应照搬 AgentScope app，而应补强两点：workspace 一等边界、permission context 与 workspace root 绑定。

## AgentScope Java

**核心亮点：AgentCard / A2A / Nacos / Spring Boot 的企业服务化链路。**

AgentScope Java 是独立项目，不是 Python AgentScope 的子目录或“2.x 同仓”。它把 agent 作为 Java/Spring 服务暴露：`AgentScopeA2aServer` 提供 AgentCard endpoint 与 JSON-RPC/SSE 调用入口，`NacosA2aRegistry` 发布 AgentCard，`NacosAgentCardResolver` 订阅并缓存更新，Spring Boot starter 自动装配 controller 和 ready listener。`ReActAgent` 还按 `(userId, sessionId)` 维持 state/permission slot，同一 slot 串行、不同 session 并行。

**适用范围：**

适合企业 Java 基建、已有 Nacos/Spring Boot 的多 agent 服务化部署，尤其是需要跨系统 agent 互操作、AgentCard 能力契约和 A2A 协议边界的场景。

**KB 维护建议：**

已维护 `wiki/_impl/agent-registry-discovery--agentscope-java.md`，并把 L1 `agent-registry-discovery` 的 AgentCard/A2A/Nacos 参考源从 Python AgentScope 改为 `agentscope-java`。

**agent-os 借鉴：**

agent-os 的 registry 应分两档：P1 用 Postgres/Redis/k8s Service 做轻量 worker registry；P2/P3 再引入 AgentCard/A2A/Nacos 风格，用于跨语言、跨组织、跨系统 agent 互操作。

## DeerFlow

**核心亮点：生产运行时硬化与可回放评估。**

DeerFlow 最值得看的不是某个 memory 模块，而是工程化运行时：`RunManager` 管 run 生命周期并对 SQLite 短暂锁冲突做 bounded retry；`JsonlRunEventStore` 用 per-thread lock + JSONL 文件做轻量单节点事件持久化；tool output middleware 将大输出外置到文件并给模型 preview/ref；checkpointer/provider 把 LangGraph 状态持久化接入生产部署；`subagent_status` 用结构化字段替代前端字符串匹配。

本轮新增 commit `63ce88f8` 改进 replay fixture key：从 conversation-only hash 变成 caller + conversation hash，避免 lead agent、title middleware、suggest agent 等不同 caller 复用同一个 replay bucket。这是很实用的评估稳定性设计。

**适用范围：**

适合 Web/Gateway 型长任务 agent，需要 run history、SSE/流式恢复、E2E replay、工具输出预算和测试稳定性的场景。

**KB 维护建议：**

更新 `session-recovery--deer-flow`、`evaluation-observability--deer-flow`、`tool-system--deer-flow`。新增或扩展 pattern：caller-aware replay key、run-event JSONL store、tool-output externalization。

**agent-os 借鉴：**

agent-os 已有 TaskStore/continuation，但缺少 DeerFlow 这种 run event 持久化和 replay fixture 体系。建议 P1 增加 append-only run event store，并在评估测试中引入 caller-aware replay key。

## Hermes Agent

**核心亮点：平台化 agent shell 和 provider/gateway 适配。**

Hermes 的价值在“把 agent 作为跨 provider、跨 gateway、跨本地/远程文件的产品化 shell”。它有 provider fallback chain、recommended model catalog 的磁盘缓存、gateway/TUI server、`file.attach` 到 workspace 的引用语义、memory provider manager，以及 `<memory-context>` fencing + streaming scrubber，避免 memory 注入泄露到输出。

**适用范围：**

适合多 provider、多入口渠道、需要远程 gateway 和本地 CLI/TUI 共用同一 agent core 的产品型 agent。

**KB 维护建议：**

更新 `channel-remote--hermes-agent`、`memory-system--hermes-agent`、`prompt-system--hermes-agent`。但 UI/桌面细节不应过度进入 L1，重点抽“provider fallback”和“memory fencing”。

**agent-os 借鉴：**

agent-os 已有 provider abstraction、ASGI/SSE/A2A。可借鉴 Hermes 的 provider fallback 配置、model catalog cache、memory-context 输出 scrubber。优先级低于 Tencent/DeerFlow，因为 agent-os 当前更缺证据链 memory 和 run replay。

## OpenHarness

**核心亮点：SDK runtime 与个人 agent 应用层分离。**

OpenHarness 同时包含 OpenHarness runtime 和 `ohmo` 个人 agent 应用。`ohmo/` 是与 `src/openharness` 并列的产品层：独立 workspace、memory、skills、plugins、gateway、session backend，但底层复用 OpenHarness runtime。这是很好的边界示范：SDK 提供 engine、commands、memory scan、channel、tasks；产品 app 只注入 persona/system prompt、个人 memory 和 channel policy。

记忆层也有明显进化：memory frontmatter schema、signature 去重、TTL、scope/type、team memory secret check、`MEMORY.md` bounded entrypoint、auto-dream consolidation 的 lock/backup/rollback/diff。

**适用范围：**

适合“框架 + 自家 agent 产品”同时演进的项目，尤其要避免把产品人格、渠道规则、个人 memory 污染 SDK 核心。

**KB 维护建议：**

更新 `memory-system--openharness`、`channel-remote--openharness`、`hooks--openharness`。可抽 pattern：SDK core 与 product agent shell 分层。

**agent-os 借鉴：**

agent-os 应保持 runtime SDK 边界，不把特定个人 agent 行为塞进 core。可借鉴 OpenHarness 的 structured memory schema、auto-dream 的 backup/rollback/preview 机制，以及 app-level workspace 注入。

## MiroFish

**核心亮点：群体仿真与数字孪生 agent 应用。**

MiroFish 不是通用 agent runtime，而是“多智能体社会仿真”产品：通过 Zep GraphRAG 管用户/关系/历史图谱，用 OASIS 生成 Twitter/Reddit 双平台行为，用 ReportAgent 汇总仿真结果。它的价值在应用范式，而不是通用 SDK 抽象。

**适用范围：**

适合社交网络仿真、数字孪生、群体行为分析、舆情推演。对代码 agent/runtime agent 的直接借鉴较少。

**KB 维护建议：**

保留在 `multi-agent` 的应用型案例即可，不建议为 agent-os core 大量吸收。可以补一页“digital twin swarm simulation”型 L2/insight。

**agent-os 借鉴：**

P3。只有当 agent-os 要支持群体仿真或模拟任务时，再借鉴它的 profile generation、GraphRAG 状态和 report agent pipeline。

## MemPalace

**核心亮点：memory 系统的插件契约和一致性测试。**

旧 KB 把 MemPalace 描述为“ChromaDB raw verbatim memory”，但 v3.4.0 已经明显进化为可扩展 memory 平台。`backends/base.py` 定义 typed `BaseCollection` / `BaseBackend` / `PalaceRef` / `QueryResult` / `GetResult`；`backends/registry.py` 用 entry points 发现第三方 backend；conformance tests 明确验证 palace id / namespace isolation；RFC 002 又把 source adapter 的 declared transformations、privacy class、incremental ingest、metadata schema 正式化。当前 wiki 已维护 `memory-system--mempalace`，并在 `_index.md` 中列为 1 个 L2。

**适用范围：**

适合长期 memory 平台、企业多源 ingest、多 backend 可替换、需要明确隔离契约和 conformance suite 的项目。

**KB 维护建议：**

`wiki/_impl/memory-system--mempalace.md` 已重写，不应再以 Chroma raw memory 为唯一主线。

**agent-os 借鉴：**

P1。agent-os 的 memory backend/store 可以借鉴 MemPalace 的 backend contract、capability flags、entry-point plugin、conformance tests，尤其是 isolation contract。

## SimpleMem

**核心亮点：统一 memory router 和离线检索策略优化。**

SimpleMem v0.3.0 把之前割裂的 text/omni 系统统一到 `SimpleMem` / `create(mode=...)` / `AutoMemory` router。text backend 仍强调语义结构化压缩、semantic/lexical/symbolic 多视图检索；omni backend 负责多模态；`simplemem.optimize()` 暴露 EvolveMem 的降级版离线优化入口，用 dev questions 搜索 retrieval hyperparameters。

**适用范围：**

适合 memory retrieval 策略实验、离线优化检索配置、文本 memory 与多模态 memory 都要覆盖但不想暴露两个不兼容入口的场景。

**KB 维护建议：**

当前 `shelf/simplemem/SHELF.md` 的归档理由已过期：v0.3.0 已经解决“接口不兼容”的核心问题。是否升入 wiki 另说，但至少应更新 shelf 状态，并把 `optimize()` / EvolveMem 离线检索优化作为候选 insight。

**agent-os 借鉴：**

P1。agent-os 可以先不做完整 SimpleMem memory，但很值得引入“离线 retrieval policy optimizer”：用开发集评估 top_k、fusion weights、context budget、query decomposition 等配置，生成可部署 memory config。

## TencentDB-Agent-Memory

**核心亮点：可追溯的符号化短期 memory + 分层长期 memory。**

TencentDB-Agent-Memory 是本轮新增项目，值得纳入 KB。它有两条主线：

1. 短期 memory：把大型 tool outputs 外置到 `refs/*.md`，中层用 `offload-<session>.jsonl` 记录 tool_call、summary、result_ref、tool_call_id、score，顶层用 Mermaid MMD 图作为 LLM 可见 canvas。模型只看高密度符号图，需要细节时通过 `node_id` drill-down 到原始证据。
2. 长期 memory：L0 Conversation 原始 JSONL → L1 Atom 原子事实 → L2 Scenario 场景块 → L3 Persona 用户画像。上层负责日常注入，下层保留证据链。

它的工程实现也有价值：host-neutral `RuntimeContext` / `HostAdapter` / `LLMRunnerFactory`，OpenClaw/Hermes/Gateway 可复用同一 memory core；JSONL 有 sanitize/validation/容错解析；L3 压缩分 mild/aggressive/emergency 三档，且对 current task 做保护。

**适用范围：**

适合长任务、工具输出巨大、需要保留可审计证据链、且要跨 session 学习用户偏好/工作流的 agent。

**KB 维护建议：**

新增 source 条目。至少写入：

- `wiki/_impl/context-management--tencentdb-agent-memory.md`
- `wiki/_impl/memory-system--tencentdb-agent-memory.md`
- 可抽 pattern：`symbolic-offload-drilldown` 或 `layered-memory-with-evidence-chain`

**agent-os 借鉴：**

P0。agent-os 当前已有 cap+nudge 和 compressed-history recall，但没有“原始工具结果 refs + JSONL summary + 图节点 drill-down”的一等抽象。建议把 Tencent 的方案作为 cap+nudge 的补充路径：默认幂等廉价工具继续 cap+nudge；非幂等、昂贵、一次性、需要审计的大结果走 optional `ToolResultEvidenceStore`，写 raw ref + summary + node_id，再在 context projection 中渲染轻量索引。

## 对 agent-os 的优先级建议

| 优先级 | 建议 | 主要来源 | 理由 |
|---|---|---|---|
| P0 | 增加可选工具结果证据链：raw refs、summary JSONL、node_id、recall/drill-down。 | TencentDB-Agent-Memory、DeerFlow | 弥补 cap+nudge 对非幂等/昂贵工具的不足，同时保留 Claude Code 式默认低 token 成本。 |
| P0 | 明确 cap+nudge 与 evidence offload 的适用边界。 | agent-os 现有 spec、TencentDB-Agent-Memory | 幂等廉价工具不需要 stash；不可重放或审计场景需要 stash。 |
| P1 | Run event store + caller-aware replay fixture。 | DeerFlow | 支撑 agent-os 服务化后的可回放测试、评估和线上诊断。 |
| P1 | Memory backend/source adapter contract + conformance tests。 | MemPalace | 让 memory 后端、source ingest、tenant isolation 有可验证契约。 |
| P1 | 离线 retrieval policy optimizer。 | SimpleMem | 用 dev set 优化 memory recall，不把 retrieval 策略写死。 |
| P1 | SDK core 与产品 agent shell 分层。 | OpenHarness / ohmo | 防止产品人格、个人 workspace、渠道策略污染 agent-os runtime SDK。 |
| P1 | Workspace + permission context 一体化。 | AgentScope | 面向多用户/远程执行时，workspace 不是目录参数，而是权限和审计边界。 |
| P1 | Registry 抽象分层：轻量 worker registry 与 AgentCard/A2A 企业互操作分开。 | AgentScope Java | 避免 P1 过早上 Nacos，同时为 P2/P3 跨系统 agent 互联预留契约。 |
| P2 | Provider fallback / model catalog cache / memory scrubber。 | Hermes Agent | 增强平台化体验，但不是当前最大缺口。 |
| P3 | 群体仿真 / 数字孪生 pipeline。 | MiroFish | 应用型能力，除非 agent-os 明确进入仿真场景。 |

## 对 ai-knowledge 的维护建议

1. AgentScope：把 Python 2.x 的 session/workspace/message-bus/team runtime 作为主线，更新 `multi-agent`、`channel-remote`、`sandbox-isolation` 相关 L2。
2. AgentScope Java：已新增独立 source，维护 `agent-registry-discovery--agentscope-java`，并把 AgentCard/A2A/Nacos 从 Python AgentScope 旧结论中迁出。
3. TencentDB-Agent-Memory：新增 source，优先写 context-management 和 memory-system L2，再抽 evidence-chain pattern。
4. MemPalace：已更新 `memory-system--mempalace`，从 raw Chroma memory 改为 backend/source adapter contract 主线，并补充当前 30 个 MCP 工具、BM25 混合检索和完整响应分块事实。
5. SimpleMem：更新 shelf 状态，记录 v0.3.0 unified router 与 EvolveMem optimizer。
6. DeerFlow：补 run event store、caller-aware replay、tool output externalization。
7. OpenHarness：补 SDK/product split、structured memory schema、auto-dream consolidation。
8. Hermes：补 provider fallback、memory fencing、gateway file ref。
9. MiroFish：保持应用型 multi-agent 案例，不宜提升为 agent-os core 参考。
