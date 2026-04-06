---
title: Session Recovery
aliases: [会话恢复, checkpoint, fault tolerance]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[runtime-state]]"
    type: uses
  - target: "[[query-loop]]"
    type: extends
sources: [claude-code, openharness]
---

## 一句话定义

Agent 断了怎么办 — checkpoint 保存、状态恢复、容错机制。

## 核心问题

- Checkpoint 保存什么（全部状态 vs 增量）？
- 恢复时怎么判断从哪里继续？
- 网络断开、进程崩溃、用户中断分别怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | 恢复"工作现场"而非重放聊天记录：将 file history、attribution state、context collapse 状态、worktree session、agent type、cost state 等完整 runtime state 持久化，跨进程重启后从中断点继续工作 | 以纯 JSON 文件为存储介质，每个 user turn 结束后自动快照完整会话状态（含完整 messages 列表、usage 统计、80 字符摘要），支持 `--continue`（恢复最近）和 `--resume`（恢复指定）两个 CLI 标志；整个实现仅 178 行 |
| 关键特点 | 交互模式与 headless 模式共享同一套恢复逻辑（`processResumedConversation` 统一协调）；worktree 恢复用 `process.chdir()` 做 TOCTOU-safe 存在性检查；fork session 时提前 seed content-replacement 记录防止 tool_use_id 匹配失败 | 人类可读存储（纯 JSON，无 SQLite、无二进制格式，可直接用文本编辑器查看）；天然可移植（JSON 文件可直接拷贝跨机器迁移）；`export_session_markdown()` 支持将会话导出为 Markdown 归档 |
| 局限 | context collapse 恢复依赖 feature flag；coordinator 模式不匹配只报 warning 不强制中断；worktree 被删除后静默降级不告知用户 | Session ID 每次调用生成新 uuid，无法跨生命周期保持稳定标识；仅 turn 间快照，turn 执行中途崩溃无法恢复；无命名会话，只能通过 id 或"最近"定位 |

## 设计权衡

### 方案对比

| 方案 | 核心机制 | 存储开销 | 实现复杂度 | 恢复粒度 |
|------|---------|---------|-----------|---------|
| 无恢复（Stateless） | 不持久化任何状态 | 零 | 极低 | 无（重头开始） |
| 对话历史快照（Conversation Snapshot） | 每轮结束后 JSON dump 完整消息列表 | 线性增长 | 低（178 行可实现，OpenHarness 案例） | Turn 边界 |
| 状态检查点（State Checkpoint） | 定期序列化完整 agent runtime state | 较大（含 worktree、tool state） | 高（需定义"完整状态"的边界） | 任务执行中途 |
| 事件溯源（Event Sourcing） | append-only 事件日志，支持任意时间点 replay | 最大（所有事件累积） | 最高（事件 schema 版本管理） | 任意事件粒度 |

### 场景决策指南

**API endpoint / serverless 函数** → 无恢复。每次调用无状态，崩溃代价低，重试即可。加入恢复逻辑只会增加冷启动延迟。

**开发助手 / CLI 工具**（如 OpenHarness）→ 对话历史快照。用户期望 `--continue` 能回到上次对话，实现简单（纯 JSON 文件），人类可读可移植。注意：仅能在 turn 边界恢复，turn 执行中途崩溃不可恢复。

**自动化工作流 / 长任务 agent**（如 Claude Code 复杂任务）→ 状态检查点。任务跑几十分钟期间进程可能因网络或资源问题崩溃，必须能从中断点续跑而不是重头来过。关键：检查点要包含 worktree 路径、tool execution state、cost state，仅保存消息历史不够。

**合规系统 / 需要审计追溯** → 事件溯源。每一步工具调用、模型输出都必须可追溯、可重放、可对账。接受存储和 replay 延迟的代价。

### 常见陷阱

**恢复后上下文已过期**：快照保存时引用的文件、环境变量在恢复时可能已变更。Claude Code 用 `process.chdir()` 做 TOCTOU-safe 检查；裸 JSON 快照方案完全没有这一步，恢复后 agent 可能基于陈旧状态继续操作，产生错误行为。解法：恢复后先验证关键外部状态是否与快照一致。

**快照中含敏感信息**：消息历史里可能有用户粘贴的 API key、数据库密码、个人信息。直接 JSON dump 到磁盘意味着这些明文存在文件系统。解法：存储前加密，或对 messages 中的已知模式做脱敏处理。

**快照文件无限堆积**：每个 session 一个文件，长期运行后目录膨胀。OpenHarness 没有 TTL 或清理机制。解法：设置保留策略（如最近 N 个或 N 天），或用 SQLite 做存储层统一管理生命周期。

**检查点包含不可序列化对象**：State Checkpoint 方案的核心难点——文件句柄、子进程引用、事件监听器无法直接序列化。Claude Code 的做法是只持久化可序列化的 ID 和路径，恢复时重新建立连接，而不是试图序列化活跃对象。

## L2 详情

- [[session-recovery--claude-code]]
- [[session-recovery--openharness]]
