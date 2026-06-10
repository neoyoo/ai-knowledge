---
title: Session Recovery
aliases: [会话恢复, checkpoint, fault tolerance]
category: L1
created: 2026-04-06
updated: 2026-06-10
relations:
  - target: "[[runtime-state]]"
    type: uses
  - target: "[[query-loop]]"
    type: extends
  - target: "[[tool-system]]"
    type: depends_on
    note: "AgentScope 2.x 将工具组激活状态保存到 AgentState.tool_context.activated_groups；session 恢复后 Toolkit 每轮重建，因此完整恢复依赖 tool-system 的稳定工具组定义"
  - target: "[[memory-system]]"
    type: depends_on
    note: "AgentScope 2.x 没有内置跨会话长期 memory；SessionRecord.state 中的 summary/context/tool cache 只承担 session-local recovery，长期 memory 需要外接系统"
sources: [claude-code, openharness, deer-flow, hermes-agent, agentscope]
---

## 一句话定义

Agent 断了怎么办 — checkpoint 保存、状态恢复、容错机制。

## 核心问题

- Checkpoint 保存什么（全部状态 vs 增量）？
- 恢复时怎么判断从哪里继续？
- 网络断开、进程崩溃、用户中断分别怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent | AgentScope |
|------|------------|-------------|----------|-------------|------------|
| 核心设计 | 恢复"工作现场"而非重放聊天记录：将 file history、attribution state、context collapse 状态、worktree session、agent type、cost state 等完整 runtime state 持久化，跨进程重启后从中断点继续工作 | 以纯 JSON 文件为存储介质，每个 user turn 结束后自动快照完整会话状态（含完整 messages 列表、usage 统计、80 字符摘要），支持 `--continue`（恢复最近）和 `--resume`（恢复指定）两个 CLI 标志；整个实现仅 178 行 | LangGraph checkpointer 负责 ThreadState 持久化；Gateway/runtime 在其上叠加 run lifecycle 控制：run 前捕获 checkpoint snapshot，cancel 时 interrupt + rollback，worker/manager 处理 checkpoint restore/delete 与 orphaned inflight run reconciliation | 以单文件 SQLite WAL 为核心存储，将消息、推理链、成本统计全部结构化持久化；最大特色是**压缩触发的 session 分裂**——`_compress_context()` 将旧 session 标记为 `ended`、生成新 session ID，并通过 `parent_session_id` 外键维系链式谱系；Gateway 层提供可配置的 `SessionResetPolicy`（none/idle/daily/both），对多平台（Telegram/Discord/Slack 等）实现差异化生命周期管理 | 服务化 runtime 恢复模型：`StorageBase` 持久化 `SessionRecord.state: AgentState`，`ChatService` 每轮从 storage/workspace/model/tool managers 重建 agent，`MessageBus.session_run()` 用 session 锁串行化执行并提供 SSE replay/live fan-out |
| 关键特点 | 交互模式与 headless 模式共享同一套恢复逻辑（`processResumedConversation` 统一协调）；worktree 恢复用 `process.chdir()` 做 TOCTOU-safe 存在性检查；fork session 时提前 seed content-replacement 记录防止 tool_use_id 匹配失败 | 人类可读存储（纯 JSON，无 SQLite、无二进制格式，可直接用文本编辑器查看）；天然可移植（JSON 文件可直接拷贝跨机器迁移）；`export_session_markdown()` 支持将会话导出为 Markdown 归档 | Per-step checkpoint 细于 per-turn；三档 checkpointer 无缝切换；Postgres 支持横向扩展；run cancel/rollback 有明确 API 语义，并通过 E2E 测试验证可回到 run 前状态 | CLI resume 两阶段加载（`_preload_resumed_session()` 提前展示历史，`_init_agent()` 检测到 `conversation_history` 非空则跳过重复 DB 查询）；v6 schema 新增 `reasoning / reasoning_details / codex_reasoning_items` 三列，保证推理型 provider 恢复后多轮推理上下文连续；FTS5 全文索引自动随写入同步，支持跨会话历史检索 | Replay log 与持久消息分工清楚：MessageBus 负责 in-flight events/replay/cancel/inbox/wakeup，Storage 负责 SessionRecord/message history；agent 对象不跨请求存活，只保存可恢复 state 和低频 config；Continuation event 通过 `reply_id` 支持待确认/外部执行结果继续 |
| 局限 | context collapse 恢复依赖 feature flag；coordinator 模式不匹配只报 warning 不强制中断；worktree 被删除后静默降级不告知用户 | Session ID 每次调用生成新 uuid，无法跨生命周期保持稳定标识；仅 turn 间快照，turn 执行中途崩溃无法恢复；无命名会话，只能通过 id 或"最近"定位 | Checkpoint 不透明（序列化格式不可读，调试困难）；无人类可读导出；存储无限增长（无自动 TTL 或清理）；跨 checkpointer 迁移难；rollback 语义依赖 run lifecycle 层正确接管 cancel | **轮末刷写**（非每步追加）：`_flush_messages_to_session_db()` 在 turn 结束时批量写入，进程在 tool 执行中途被 kill 会丢失当轮全部消息；压缩分裂失败不回滚，旧 session 已被 `end_session` 但新 session 未创建，后续写入会进入错误 session；gateway reset policy 状态依赖进程内 `sessions.json`，不在 SQLite 中，进程重启后需重新加载 | `AgentState` 是粗粒度 state，缺少 field-level diff；恢复依赖工具/workspace/model 配置可重建，部署漂移会让旧 session 无法完整继续；MessageBus replay log 是短期事件流，不替代长期事件溯源；本地 workspace manager 的路径隔离弱于 Web 多租户要求 |

## 设计权衡

### 方案对比

| 方案 | 核心机制 | 存储开销 | 实现复杂度 | 恢复粒度 |
|------|---------|---------|-----------|---------|
| 无恢复（Stateless） | 不持久化任何状态 | 零 | 极低 | 无（重头开始） |
| 对话历史快照（Conversation Snapshot） | 每轮结束后 JSON dump 完整消息列表 | 线性增长 | 低（178 行可实现，OpenHarness 案例） | Turn 边界 |
| 结构化关系型存储（Structured DB） | SQLite 持久化消息+元数据+推理链，支持全文检索和谱系查询 | 可控（WAL checkpoint 防膨胀） | 中（schema 版本管理，双轨写入协调） | Turn 边界（轮末 flush） |
| 状态检查点（State Checkpoint） | 定期序列化完整 agent runtime state | 较大（含 worktree、tool state） | 高（需定义"完整状态"的边界） | 任务执行中途 |
| 事件溯源（Event Sourcing） | append-only 事件日志，支持任意时间点 replay | 最大（所有事件累积） | 最高（事件 schema 版本管理） | 任意事件粒度 |

### 场景决策指南

**API endpoint / serverless 函数** → 无恢复。每次调用无状态，崩溃代价低，重试即可。加入恢复逻辑只会增加冷启动延迟。

**开发助手 / CLI 工具**（如 OpenHarness）→ 对话历史快照。用户期望 `--continue` 能回到上次对话，实现简单（纯 JSON 文件），人类可读可移植。注意：仅能在 turn 边界恢复，turn 执行中途崩溃不可恢复。

**多平台 chatbot / 长期运营的对话 agent**（如 Hermes Agent）→ 结构化关系型存储 + 可配置 reset policy。需要同时服务 Telegram/Discord/Slack 等平台，各平台空闲周期和重置需求不同，`SessionResetPolicy`（none/idle/daily/both）可按平台或会话类型独立配置。FTS5 全文检索让 agent 能追溯跨会话的历史决策依据。context 压缩时用 `parent_session_id` 链构建谱系而非破坏性截断，用户体验上一个项目可无缝跨越多次压缩。

**自动化工作流 / 长任务 agent**（如 Claude Code 复杂任务）→ 状态检查点。任务跑几十分钟期间进程可能因网络或资源问题崩溃，必须能从中断点续跑而不是重头来过。关键：检查点要包含 worktree 路径、tool execution state、cost state，仅保存消息历史不够。

**Web/Gateway 长任务 agent**（DeerFlow 新版）→ checkpoint + run lifecycle 控制层。底层 checkpoint 只能回答“状态存在哪里”；产品还必须定义“取消 run 后回滚到哪一个 checkpoint”“orphaned inflight run 如何 reconciliation”“HTTP cancel 返回什么语义”。agent-os 如果同时支持本地代理和 Web 分布式代理，应把 `CheckpointStore` 与 `RunController` 拆成两个 ABC，分别实现 local 与 Web/distributed 版本。

**合规系统 / 需要审计追溯** → 事件溯源。每一步工具调用、模型输出都必须可追溯、可重放、可对账。接受存储和 replay 延迟的代价。

**多 agent Web runtime（AgentScope Python 2.x）** → `SessionRecord + MessageBus + wakeup`。team worker 是有独立 `AgentRecord` / `SessionRecord` / `AgentState` 的 hidden agent；leader 发消息时先写 inbox，再 enqueue wakeup。它不是本地对象树快照，而是把 worker 纳入同一套 session lock、event stream、workspace 和 storage 基础设施。agent-os 如果要同时支持本地代理和 Web 分布式 agent，应把 `SessionStore`、`MessageBus`、`WorkspaceManager`、`RunController` 拆成 ABC，再分别实现 local 与 distributed 版本。

**SaaS 多租户 / 同一 session 可能被多个 worker 触发** → session lock + replay/live event bus。AgentScope 2.x 的 `MessageBus.session_run(session_id)` 在运行边界加互斥锁，Redis 实现用 token/heartbeat 防止并发运行同一 session；SSE endpoint 先 replay buffered events 再 live subscribe，前端刷新或重连不会丢当前 run 的事件。注意：replay log 是短期 buffer，长期恢复仍依赖 Storage 的 message/session state。

### 常见陷阱

**恢复后上下文已过期**：快照保存时引用的文件、环境变量在恢复时可能已变更。Claude Code 用 `process.chdir()` 做 TOCTOU-safe 检查；裸 JSON 快照方案完全没有这一步，恢复后 agent 可能基于陈旧状态继续操作，产生错误行为。解法：恢复后先验证关键外部状态是否与快照一致。

**快照中含敏感信息**：消息历史里可能有用户粘贴的 API key、数据库密码、个人信息。直接 JSON dump 到磁盘意味着这些明文存在文件系统。解法：存储前加密，或对 messages 中的已知模式做脱敏处理。Hermes Agent 在 gateway 层对 WhatsApp/Signal/Telegram 等平台的 user_id 做 SHA-256 截断哈希，PII 不进入 LLM context，是一种实用的分层脱敏策略。

**快照文件无限堆积**：每个 session 一个文件，长期运行后目录膨胀。OpenHarness 没有 TTL 或清理机制。解法：设置保留策略（如最近 N 个或 N 天），或用 SQLite 做存储层统一管理生命周期。SQLite WAL 方案（Hermes）通过每 50 次写操作触发 PASSIVE checkpoint 控制 WAL 文件大小，但消息表本身仍无限增长——需要在应用层实现 TTL 或归档策略。

**检查点包含不可序列化对象**：State Checkpoint 方案的核心难点——文件句柄、子进程引用、事件监听器无法直接序列化。Claude Code 的做法是只持久化可序列化的 ID 和路径，恢复时重新建立连接，而不是试图序列化活跃对象。

**轮末 flush 与崩溃丢失**：Hermes 和 OpenHarness 均在 turn 边界写入，若进程在 tool 执行中途被 kill（SIGKILL），当轮全部消息（包括已完成的 tool call/result）会丢失。Claude Code 通过每步追加写入 transcript 文件解决了这个问题。选择轮末 flush 要清楚接受的代价：实现简单但无法防御中途崩溃。

**压缩触发分裂的原子性问题**：Hermes 的 `_compress_context()` 先调用 `end_session(old_id)` 再创建新 session；若新 session 创建失败，旧 session 已被标记结束，后续 flush 仍用旧 session ID 写入——消息会进入已结束的 session，且 `_last_flushed_db_idx = 0` 重置意味着全量重写而非增量追加。解法：将 end+create 包在同一事务中，或在失败时回滚旧 session 的 `ended_at`。

**schema 版本演进中的推理链兼容性**：Hermes v6 新增 `reasoning / reasoning_details / codex_reasoning_items` 三列是为了支持推理型 provider（OpenRouter/Nous）的多轮推理上下文重建。如果存储层没有这类字段，这些 provider 在 session 恢复后会丢失推理链，导致表现退化。使用推理型模型的 agent 需要在设计存储 schema 时提前预留推理链字段。

**把 event replay 当长期持久化**：AgentScope 2.x 的 MessageBus replay log 服务 SSE 重连和 live fan-out，锁退出后会 trim；完整历史在 Storage message record，恢复状态在 SessionRecord.state。自建系统若把短期 event buffer 当作唯一事实来源，进程重启或 trim 后会丢审计线索。解法：明确 `EventBus`、`MessageStore`、`CheckpointStore` 三者职责。

## 相关模式

- [[state-module-tree-serialization]] — 历史 AgentScope 1.x 模式；2.x 已迁移到 `AgentState + SessionRecord + MessageBus`

## L2 详情

- [[session-recovery--claude-code]]
- [[session-recovery--openharness]]
- [[session-recovery--deer-flow]]
- [[session-recovery--hermes-agent]]
- [[session-recovery--agentscope]]
