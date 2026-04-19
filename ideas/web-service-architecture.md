---
title: Web 服务化 Agent 架构想法
domain: web-service-architecture
updated: 2026-04-19
---

# Web 服务化 Agent 架构想法

从 2026-04-19 trip-os "上下文注入 + 历史加载"决策的 KB-first 摩擦挤出来的多条想法。核心目标：让 KB 对"把 agent 从 CLI 升级到 Web 服务"这条路径有完整指南。

---

## 2026-04-19 — web-request-lifecycle-for-agent（跨 4 概念 pattern）

**status**: inbox  
**potential_target**: `wiki/_patterns/web-request-lifecycle-for-agent.md`

**核心想法**：Web 版本 agent 的 HTTP 请求生命周期是**跨 4 个 L1 概念的协同**，不归任一个单管：

```
[Auth 中间件]
     ↓
[Session 加载]   ← session-recovery   (从 storage → 内存 Session.messages)
     ↓
[Context 整理]   ← context-management (超预算则折叠/压缩/截断)
     ↓
[Prompt 组装]    ← prompt-system      (静态 + 动态区块注入)
     ↓
[Agent.run]     ← query-loop
     ↓
[流式 + 增量持久]                      (SSE 边发边写)
```

**为什么值得**：trip-os 马上要改 Web 版本，这个生命周期图是最直接要的指导；目前 KB 各家对比零散在 4 个 L1 页里，没有综合视角；是可迁移的架构模式（任何 agent 做 Web 服务都要走这条路）。

**满足 pattern 收录四标准**：跨概念 ✅（4 个）/ 可迁移 ✅ / 参考实现 🟡（neoagent FastAPIChannel + Hermes _agent_cache）/ 具体问题 ✅。

**关键决策点**（pattern 页内容骨架）：
- **Session 加载显式 vs 隐式**：显式胜（Claude Code `processResumedConversation` 模式）
- **Agent 实例缓存**：按 session key 缓存（Hermes `_agent_cache` 模式，**防 prompt cache 失效、省 10x token**）
- **History 分段加载**：按 max_turns 配额 load，超出走 context-management 折叠
- **增量持久 vs 批量**：SSE 流式要求边发边持久
- **恢复语义**：恢复"工作现场"而非"重放聊天记录"

**参考实现**：
- neoagent `FastAPIChannel` + `Session` + `JsonFileStorage`（shelf）——骨架现成
- DeerFlow middleware pipeline 五切入点（before/after model, agent, wrap_tool_call）
- Hermes `_agent_cache` 按 session key 缓存

**落地原则**：**不等参考实现不写**。等 trip-os 真跑起 Web 版本（把 neoagent JsonFileStorage 换成 Postgres/ES/MinIO），再据实写 pattern 页。

**相关**：`wiki/channel-remote.md`、`wiki/session-recovery.md`、`wiki/prompt-system.md`、`wiki/context-management.md`、`projects/trip-os/kb-friction.md`

---

## 2026-04-19 — prompt cache 友好的 system prompt 结构

**status**: inbox  
**potential_target**: `wiki/prompt-system.md` 扩充章节 或 `wiki/_insights/prompt-cache-stable-structure.md`

**核心想法**：prompt-system 的设计对 prompt cache 命中率影响巨大。要做 Web 版本需要搞懂"什么设计让 Anthropic/OpenAI 的 prefix cache 能命中"。**同一 session 的 agent 实例必须缓存**（否则每次新建 agent、重组装 prompt、cache 命中率崩盘，费用 ~10x）。

**为什么值得**：Hermes 在 `wiki/channel-remote.md` 的 `_agent_cache` 对比里点破了这个陷阱，但**没有正面专题讲"prompt cache 友好的 prompt 结构怎么设计"**——什么能变（放在 cacheable boundary 之后）、什么不能变（放 boundary 之前）、动态内容如何注入而不破坏 cache。AgentScope 的 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 概念是信号但没展开。

**内容骨架**：
- 主流 provider 的 prefix caching 机制（Anthropic / OpenAI / Bedrock）
- Boundary 设计：静态段 vs 动态段的切分点
- 常见陷阱：时间戳、freed 清单、可用工具列表这些动态内容该放哪
- cache 失效的触发条件清单
- 监控：怎么观测 cache hit rate

**落地**：先作为"见过的坑"写 insight 卡；如果跨项目对比够丰富再升级为 L1 pattern。

**相关**：`wiki/channel-remote.md` Hermes `_agent_cache` 讨论、`wiki/prompt-system.md` AgentScope DYNAMIC_BOUNDARY

---

## 2026-04-19 — SSE 流式 + 增量持久化策略

**status**: inbox  
**potential_target**: `wiki/session-recovery.md` 扩充 或 `cookbook/tools/patterns/streaming-incremental-persist.md`

**核心想法**：Web 版本 + SSE 流式响应场景下，agent 每产一个 chunk 就应该增量持久化到 storage，而不是等整次 response 完批量写。理由：客户端断网时从 storage 恢复体验更好；token 费用已经产生，别因为崩溃白费。

**为什么值得**：`wiki/session-recovery.md` 里 OpenHarness 只讲"每个 user turn 结束自动快照"——**turn 内中途崩溃无法恢复**是明确局限。Web 场景里 turn 可能持续几分钟（subagent 递归），粒度必须更细。这是 Web 特有的要求，CLI 场景没这个紧迫性。

**内容骨架**：
- 流式场景下的持久化粒度（per-chunk / per-tool-call / per-turn）
- 回放语义：客户端重连怎么从中断点继续推送？
- 存储成本：高频写 ES / Postgres 的开销
- trade-off：一致性 vs 性能

**落地**：需要 trip-os 实测性能才有一手数据，先存着 incubate。

**相关**：`wiki/session-recovery.md`（各家局限章节）、`wiki/channel-remote.md`（DeerFlow SSE、Claude Code 远程 session）

---

## 2026-04-19 — Web 版本 agent 的存储分层架构

**status**: inbox  
**potential_target**: `cookbook/tools/patterns/storage-layering-for-web-agent.md`

**核心想法**：Web 版本要同时存：session 元数据（小、强一致）、messages（多、需检索）、artifacts（大、对象存储）。应该分层不是一锅塞。建议：

| 层 | 对应内容 | 选型 |
|---|---|---|
| 关系型 | session 元数据（user_id, session_id, created_at, status） | Postgres |
| 全文/结构检索 | messages 全量 + 用户消息搜索 | Elasticsearch |
| 对象 | artifacts（pages/media/result.json） | MinIO / S3 / 阿里云 OSS |
| 缓存 + pubsub | SSE 广播 / prompt cache 生命周期 / rate limit | Redis |

**为什么值得**：KB 完全没讨论存储分层。session-recovery 对比 5 家都没摆存储选型（各家基本是 JSON/SQLite/Postgres 混着来，没人系统性分层）。对新上 Web 版本的项目，这是躲不开的决策。

**内容骨架**：
- 四层职责边界（哪些数据去哪一层，规则）
- 跨层一致性（session 元数据更新和 messages 写入的事务边界）
- 运维 trade-off（单机 docker-compose vs k8s + 托管服务）
- 跨 region 复制考虑（如果要做）
- 参考实现：trip-os 实际部署架构（跑起来再写）

**落地**：需要 trip-os 真正跑起存储层才有一手经验，先 incubate。同 `sandbox-security.md` 的 agentscope-runtime-cookbook 一样，**抄文档的选型指南价值低**。

**相关**：`wiki/session-recovery.md`、`wiki/channel-remote.md`、`projects/trip-os/decisions/2026-04-19-web-service-security-sandbox.md`
