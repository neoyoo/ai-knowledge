---
title: Session Recovery
aliases: [会话恢复, checkpoint, fault tolerance]
category: L1
created: 2026-04-06
updated: 2026-04-08
relations:
  - target: "[[runtime-state]]"
    type: uses
  - target: "[[query-loop]]"
    type: extends
sources: [claude-code, openharness, deer-flow, hermes-agent]
---

## 一句话定义

Agent 断了怎么办 — checkpoint 保存、状态恢复、容错机制。

## 核心问题

- Checkpoint 保存什么（全部状态 vs 增量）？
- 恢复时怎么判断从哪里继续？
- 网络断开、进程崩溃、用户中断分别怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|-------------|
| 核心设计 | 恢复"工作现场"而非重放聊天记录：将 file history、attribution state、context collapse 状态、worktree session、agent type、cost state 等完整 runtime state 持久化，跨进程重启后从中断点继续工作 | 以纯 JSON 文件为存储介质，每个 user turn 结束后自动快照完整会话状态（含完整 messages 列表、usage 统计、80 字符摘要），支持 `--continue`（恢复最近）和 `--resume`（恢复指定）两个 CLI 标志；整个实现仅 178 行 | 持久化完全委托给 LangGraph checkpointer（支持内存/SQLite/Postgres 三档），每个 agent 步骤完成后自动保存 ThreadState（包含消息历史、沙箱状态、产物、todos），恢复只需传入相同 `thread_id`，agent 自身零代码实现 session recovery | 以单文件 SQLite WAL 为核心存储，将消息、推理链、成本统计全部结构化持久化；最大特色是**压缩触发的 session 分裂**——`_compress_context()` 将旧 session 标记为 `ended`、生成新 session ID，并通过 `parent_session_id` 外键维系链式谱系；Gateway 层提供可配置的 `SessionResetPolicy`（none/idle/daily/both），对多平台（Telegram/Discord/Slack 等）实现差异化生命周期管理 |
| 关键特点 | 交互模式与 headless 模式共享同一套恢复逻辑（`processResumedConversation` 统一协调）；worktree 恢复用 `process.chdir()` 做 TOCTOU-safe 存在性检查；fork session 时提前 seed content-replacement 记录防止 tool_use_id 匹配失败 | 人类可读存储（纯 JSON，无 SQLite、无二进制格式，可直接用文本编辑器查看）；天然可移植（JSON 文件可直接拷贝跨机器迁移）；`export_session_markdown()` 支持将会话导出为 Markdown 归档 | 零代码持久化（agent 逻辑完全不涉及存储细节）；per-step checkpoint（每个 tool call/model call 完成后立即保存，粒度细于 per-turn）；三档切换无缝（只改配置文件）；Postgres 支持横向扩展 | CLI resume 两阶段加载（`_preload_resumed_session()` 提前展示历史，`_init_agent()` 检测到 `conversation_history` 非空则跳过重复 DB 查询）；v6 schema 新增 `reasoning / reasoning_details / codex_reasoning_items` 三列，保证推理型 provider 恢复后多轮推理上下文连续；FTS5 全文索引自动随写入同步，支持跨会话历史检索 |
| 局限 | context collapse 恢复依赖 feature flag；coordinator 模式不匹配只报 warning 不强制中断；worktree 被删除后静默降级不告知用户 | Session ID 每次调用生成新 uuid，无法跨生命周期保持稳定标识；仅 turn 间快照，turn 执行中途崩溃无法恢复；无命名会话，只能通过 id 或"最近"定位 | Checkpoint 不透明（序列化格式不可读，调试困难）；无人类可读导出；存储无限增长（无自动 TTL 或清理）；跨 checkpointer 迁移难 | **轮末刷写**（非每步追加）：`_flush_messages_to_session_db()` 在 turn 结束时批量写入，进程在 tool 执行中途被 kill 会丢失当轮全部消息；压缩分裂失败不回滚，旧 session 已被 `end_session` 但新 session 未创建，后续写入会进入错误 session；gateway reset policy 状态依赖进程内 `sessions.json`，不在 SQLite 中，进程重启后需重新加载 |

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

**合规系统 / 需要审计追溯** → 事件溯源。每一步工具调用、模型输出都必须可追溯、可重放、可对账。接受存储和 replay 延迟的代价。

### 常见陷阱

**恢复后上下文已过期**：快照保存时引用的文件、环境变量在恢复时可能已变更。Claude Code 用 `process.chdir()` 做 TOCTOU-safe 检查；裸 JSON 快照方案完全没有这一步，恢复后 agent 可能基于陈旧状态继续操作，产生错误行为。解法：恢复后先验证关键外部状态是否与快照一致。

**快照中含敏感信息**：消息历史里可能有用户粘贴的 API key、数据库密码、个人信息。直接 JSON dump 到磁盘意味着这些明文存在文件系统。解法：存储前加密，或对 messages 中的已知模式做脱敏处理。Hermes Agent 在 gateway 层对 WhatsApp/Signal/Telegram 等平台的 user_id 做 SHA-256 截断哈希，PII 不进入 LLM context，是一种实用的分层脱敏策略。

**快照文件无限堆积**：每个 session 一个文件，长期运行后目录膨胀。OpenHarness 没有 TTL 或清理机制。解法：设置保留策略（如最近 N 个或 N 天），或用 SQLite 做存储层统一管理生命周期。SQLite WAL 方案（Hermes）通过每 50 次写操作触发 PASSIVE checkpoint 控制 WAL 文件大小，但消息表本身仍无限增长——需要在应用层实现 TTL 或归档策略。

**检查点包含不可序列化对象**：State Checkpoint 方案的核心难点——文件句柄、子进程引用、事件监听器无法直接序列化。Claude Code 的做法是只持久化可序列化的 ID 和路径，恢复时重新建立连接，而不是试图序列化活跃对象。

**轮末 flush 与崩溃丢失**：Hermes 和 OpenHarness 均在 turn 边界写入，若进程在 tool 执行中途被 kill（SIGKILL），当轮全部消息（包括已完成的 tool call/result）会丢失。Claude Code 通过每步追加写入 transcript 文件解决了这个问题。选择轮末 flush 要清楚接受的代价：实现简单但无法防御中途崩溃。

**压缩触发分裂的原子性问题**：Hermes 的 `_compress_context()` 先调用 `end_session(old_id)` 再创建新 session；若新 session 创建失败，旧 session 已被标记结束，后续 flush 仍用旧 session ID 写入——消息会进入已结束的 session，且 `_last_flushed_db_idx = 0` 重置意味着全量重写而非增量追加。解法：将 end+create 包在同一事务中，或在失败时回滚旧 session 的 `ended_at`。

**schema 版本演进中的推理链兼容性**：Hermes v6 新增 `reasoning / reasoning_details / codex_reasoning_items` 三列是为了支持推理型 provider（OpenRouter/Nous）的多轮推理上下文重建。如果存储层没有这类字段，这些 provider 在 session 恢复后会丢失推理链，导致表现退化。使用推理型模型的 agent 需要在设计存储 schema 时提前预留推理链字段。

## L2 详情

- [[session-recovery--claude-code]]
- [[session-recovery--openharness]]
- [[session-recovery--deer-flow]]
- [[session-recovery--hermes-agent]]
