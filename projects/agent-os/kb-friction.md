# agent-os KB 摩擦日志

> 真实使用 KB 过程中的摩擦记录。每条含：查询 → 去哪 → 命中度 → 缺什么。

---

## 2026-05-22 检索：agent-os SDK 客观打分基线信息

**任务**：为主线程打分 agent-os SDK 提供业界对比基线

| 时间 | 查询 | 去哪 | 命中 | 摩擦 / 缺什么 |
|---|---|---|---|---|
| 2026-05-22 | "projects/agent-os/decisions/ 关键决策" | `projects/agent-os/` | ❌ 不存在 | agent-os 项目目录在 KB 中完全缺失，没有 decisions/ 也没有 kb-friction.md |
| 2026-05-22 | "provider 抽象层对比基线（OpenAI-compatible / streaming / multimodal / extra_body）" | `wiki/` 所有文件 | ❌ 严重缺口 | KB 里没有独立的 Provider Abstraction 页面。各家做法散落在 query-loop / formatter-as-provider-boundary / prompt-system 的角落里。缺：专门对比 agentscope ModelWrapperBase vs hermes resolve_runtime_provider vs claude-code provider 架构的 L1 页 |
| 2026-05-22 | "测试 / 工程质量维度对比（test coverage / typing / lint）" | `wiki/` 所有文件 | ❌ 完全缺失 | KB 完全没有"工程质量"维度的系统对比。没有各家项目 typing 严格度、test coverage、lint 配置的横向对比 |
| 2026-05-22 | "SDK 打分 rubric（dimension / weight / 1-10）" | `wiki/` + `ideas/` + `cookbook/` | ❌ 不存在 | KB 没有现成的 agent SDK 评价 rubric 模板 |
| 2026-05-22 | "channel-agnostic agent core + thin channel adapter 模式" | `wiki/channel-remote.md` + patterns | 🟡 部分 | channel-remote.md 有"多渠道适配器"方案，formatter-as-provider-boundary 有内部消息 provider-neutral 概念，但没有专门的 pattern 页描述"channel-agnostic core + thin adapter"架构 |
| 2026-05-22 | "CLI / Web 同时支持的 SDK 模式" | `wiki/channel-remote.md` | 🟡 弱 | 有各家现状描述，但没有"怎么设计让同一 agent core 跑在 CLI stdin/stdout 和 ASGI HTTP 两种环境"的设计决策指南 |

## 2026-05-28 检索：三块生产级硬化 spec 的参考实现

**任务**：为 agent-os 写 tool-result token budget / token-aware compression+熔断 / SSE Last-Event-ID resume 三块 spec 找 KB 参考

| 时间 | 查询 | 去哪 | 命中 | 摩擦 / 缺什么 |
|---|---|---|---|---|
| 2026-05-28 | "tool-result 过大的预算/截断策略" | `wiki/_patterns/tool-metadata-driven-context-lifecycle.md` | ✅ 好 | neoagent v2 LLM-driven free/recall 四层防线 + v1 auto_free_after 反决策，直接可用 |
| 2026-05-28 | "token-aware 压缩 + 多 provider token 计数 + 失败熔断" | `wiki/context-management.md` | ✅ 好 | AgentScope 多后端 TokenCounter、Claude Code 熔断保留、Hermes 静默删除反例、tools schema 纳入计数、keep_recent token-budget 尾部，全齐 |
| 2026-05-28 | "SSE Last-Event-ID resume / 断线续传缓冲" | `wiki/channel-remote.md` + `wiki/session-recovery.md` | ❌ 严重缺口 | **对比的 5 个源全部没做好 SSE resume**：Claude Code 远程只能 HTTP 轮询不能 WS 推送、DeerFlow SSE 单向无 resume、AgentScope realtime 断开即丢无持久化。缺：流式响应的 per-event id + 服务端 replay buffer（in-memory ring vs hot store）设计指南。这是 channel-remote 的真实盲区 |

## 2026-05-28 检索：Claude Code 怎么处理大 tool result（尤其文件读取）

**任务**：判断 agent-os 的 tool-result budget 要不要做 stash+recall，对照 Claude Code 文件读取密集场景的真实做法

| 时间 | 查询 | 去哪 | 命中 | 摩擦 / 缺什么 |
|---|---|---|---|---|
| 2026-05-28 | "Claude Code Read 工具的输出限制 / 分页 / 截断行为" | `wiki/_impl/tool-system--claude-code.md` | ❌ 缺口 | tool-system 页讲了工具协议/分层池/双引擎/权限沙箱，但**完全没写工具的输出限制机制**——Read 默认 2000 行、offset/limit 分页、长行截断、大输出 cap+notice、模型据此 re-read 的模式。这是 file-heavy agent 的核心 context 控制手段，比通用压缩更前置 |
| 2026-05-28 | "tool result 过大：source re-read vs stash+recall 两种哲学对比" | wiki 全局 | ❌ 缺口 | KB 有 neoagent 的 stash+recall（free/recall 四层），但没有 Claude Code 的"工具层分页截断 + 模型 re-read"对立哲学，也没有"工具是否幂等/可廉价重读"这个决定该选哪条路的判据 |

## 2026-06-09 检索：agent-os 本地代理形态 + Web 分布式形态抽象边界

**任务**：判断 agent-os 是否应通过 ABC/Protocol 支持 local proxy agent 与 web distributed agent 两种形态，并评估 TencentDB Agent Memory 的本地 memory 借鉴价值

| 时间 | 查询 | 去哪 | 命中 | 摩擦 / 缺什么 |
|---|---|---|---|---|
| 2026-06-09 | "local proxy agent + web distributed agent runtime profile" | `wiki/channel-remote.md`、`wiki/agent-registry-discovery.md`、`wiki/multi-agent.md`、`wiki/memory-system.md`、`projects/agent-os/kb-friction.md` | 🟡 部分 | 各概念页分别覆盖 channel、registry、multi-agent、memory，但没有跨概念 pattern 描述用 LocalProfile / DistributedProfile 组合 session、registry、queue、memory、workspace、sandbox、event store 的 SDK 级架构 |
| 2026-06-09 | "TencentDB Agent Memory 是否本地文件 memory" | 新 clone 源码 `TencentDB-Agent-Memory/src/offload/*`、`src/core/store/*` | ✅ 好 | KB 尚未 ingest TencentDB Agent Memory；短期 offload 的 refs/jsonl/mmd/state.json 和长期 SQLite/TCVDB 双后端值得整理成 memory-system L2 或单源 insight |
| 2026-06-09 | "高密度符号图 memory / MMD 认知状态机是否能减少上下文并让 LLM 自主召回" | `wiki/memory-system.md`、`wiki/context-management.md`、`wiki/_patterns/tool-metadata-driven-context-lifecycle.md`、TencentDB Agent Memory 源码 | 🟡 部分 | KB 有 handle/recall 和图谱记忆相邻内容，但没有总结 TencentDB Agent Memory 的 MMD 符号图：把大量 tool result 映射为 Mermaid 节点、状态、边、node_mapping，并通过 Scene Navigation/read_file 渐进展开；需要补 insight |

## 归类

| 类 | 条目 | 说明 |
|---|------|------|
| **A. 独立 L1 页缺失** | provider-abstraction 维度 | 需要新 `wiki/provider-abstraction.md` |
| **B. 工程质量维度缺失** | test / typing / lint 横向对比 | 需要在各 L1 或新建独立页 |
| **C. Pattern 缺失** | channel-agnostic-core | 候选升入 `wiki/_patterns/` |
| **D. 评估工具缺失** | SDK 打分 rubric | 候选放 `cookbook/` |
| **E. Channel 盲区** | SSE Last-Event-ID resume / replay buffer | 5 源全缺，候选新 `wiki/_insights/` 或 channel-remote 补充 |
| **F. Pattern 缺失** | local-vs-distributed-runtime-profile | 候选升入 `wiki/_patterns/`，跨 channel-remote + registry + multi-agent + memory + sandbox |
| **G. 新源待 ingest** | TencentDB Agent Memory | 候选补 `wiki/_impl/memory-system--tencentdb-agent-memory.md` 或 `_insights/tencentdb-agent-memory--context-offload.md` |
