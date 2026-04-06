---
title: "Session Recovery — Claude Code"
category: L2
parent: "[[session-recovery]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的 session recovery 不是"重放聊天记录"，而是"恢复工作现场"——它把 file history、attribution state、context collapse 状态、todos、worktree session、model override、agent type 等 runtime state 全部持久化并在 resume 时完整重建。这种设计使得跨进程重启后，agent 可以从中断点继续工作，而不是从零开始对话。

## 架构分析

### 恢复对象：Runtime State 而非 Transcript

`sessionRestore.ts` 定义了 `ResumeResult` 类型，揭示了恢复的完整状态边界：

- `messages` — 对话历史（transcript）
- `fileHistorySnapshots` — 文件操作历史快照
- `attributionSnapshots` — git commit attribution 状态
- `contextCollapseCommits` / `contextCollapseSnapshot` — context 压缩历史
- `agentSetting` — 上次使用的 agent 类型
- `worktreeSession` — worktree 进入状态
- `mode` — coordinator/normal 模式
- `prNumber/prUrl/prRepository` — PR 关联元数据

这说明 Claude Code 把"会话"建模为一个完整的工作环境，而不仅仅是消息序列。

### 多路径统一恢复架构

Claude Code 有两条 resume 路径，但共享同一套恢复逻辑：

1. **交互模式**（REPL.tsx）：调用 `restoreSessionStateFromLog()`，挂载后异步恢复 AppState
2. **CLI 模式**（main.tsx 的 `--continue` / `--resume`）：调用 `processResumedConversation()`，在渲染前同步计算初始 state

`processResumedConversation()` 是核心协调器，按顺序处理：
1. coordinator/normal 模式匹配（避免模式不一致导致行为异常）
2. session ID 恢复（`switchSession()` + `resetSessionFilePointer()`）
3. worktree 路径恢复（`restoreWorktreeForResume()`）
4. session metadata 恢复（`restoreSessionMetadata()`）
5. context collapse 状态恢复
6. agent 定义恢复（`restoreAgentFromSession()`）
7. attribution state 计算

### Worktree Session 恢复

`restoreWorktreeForResume()` 展示了一个细节丰富的容错设计：

- 优先级：新建 worktree（`--worktree` flag）> 恢复已有 worktree
- 使用 `process.chdir()` 作为存在性检查（TOCTOU-safe）：如果目录已被删除，chdir 会抛 ENOENT
- 清除多个缓存：`clearMemoryFileCaches()`、`clearSystemPromptSections()`、`getPlansDirectory.cache.clear?.()`
- 恢复后不设置 `projectRoot`——与 `EnterWorktreeTool` 行为保持一致，让 skills/history 继续锚定原始 project

`exitRestoredWorktree()` 处理中途 `/resume` 切换 session 时的 worktree 清理，避免旧 worktree 状态污染新 session。

### Agent 定义恢复

`restoreAgentFromSession()` 处理 agent 不再可用的情况（优雅降级）：

- 若 session 记录的 agent 已不在 `activeAgents` 中，输出 debug log，清空 agent 类型，继续默认行为
- 若 agent 有绑定 model 且用户未在 CLI 指定 model，自动恢复 model override
- CLI `--agent` flag 优先于 session 记录（用户意图优先）

`refreshAgentDefinitionsForModeSwitch()` 在 coordinator/normal 模式切换时重新推导 built-in agents，避免 stale agent 定义。

### Cost State 恢复

`restoreCostStateForSession()` 在 session ID switch 后立即恢复 cost 状态，确保 `getTotalCost()` 等指标跨重启连续累积，而不是从零开始。这对长期任务的 budget 控制（`maxBudgetUsd`）至关重要。

### Fork Session 设计

`--fork-session` flag 创建新 session ID 而不重用原有 ID，但需要特殊处理：content-replacement 记录必须提前 seed 到新 session，否则 `claude -r {newSessionId}` 时 tool_use_id 找不到对应记录，导致所有 content 以 FROZEN 状态发送（prompt cache miss + token 超量）。这个细节体现了 session resume 对 content deduplication 机制的深度集成。

### 关键代码路径
- `src/utils/sessionRestore.ts` — 恢复核心逻辑，包含所有恢复函数
- `src/QueryEngine.ts` — usage 累积、cost tracking、session persistence
- `src/main.tsx` — 系统入口，startup 时根据 CLI 参数决定恢复路径
- `src/bootstrap/state.js` — session ID 管理、`switchSession()`
- `src/utils/sessionStorage.ts` — transcript 文件操作（`adoptResumedSessionFile`、`resetSessionFilePointer`）
- `src/utils/worktree.ts` — `getCurrentWorktreeSession()`、`restoreWorktreeSession()`
- `src/cost-tracker.ts` — `restoreCostStateForSession()`

## 设计亮点

- **Runtime state 完整恢复**：不只恢复消息，连 worktree 路径、git attribution、context collapse 历史都能跨重启恢复，agent 真正从"工作现场"继续
- **两路径统一设计**：交互模式（REPL）和 headless 模式（SDK/CLI）共享同一套恢复逻辑，通过 `processResumedConversation` 统一协调
- **存在性感知的 worktree 恢复**：用 `process.chdir()` 替代 `fs.access()` 做目录检查，天然 TOCTOU-safe，优雅处理 session 间被删除的 worktree
- **Fork session 的 content-replacement seeding**：在 fork 时提前写入 content-replacement 记录，避免新 session 对旧 tool_use_id 分类错误，细节工程质量高
- **Cost state 跨 session 连续性**：budget 控制不因进程重启而失准

## 局限性

- **`contextCollapseSnapshot` 需要 feature flag**：`CONTEXT_COLLAPSE` flag 控制，关闭后 context 压缩状态不会恢复，长会话的 compaction 历史丢失
- **`COMMIT_ATTRIBUTION` 是 ant-only feature**：attribution 恢复在外部用户版本中不激活
- **coordinator 模式不匹配只报 warning**：不同模式下的 agent 行为差异只产生 warning 消息，不强制中断，可能导致语义不一致
- **worktree 路径删除后降级为原始 cwd**：不会报错告知用户 worktree 已消失，静默降级可能让用户困惑

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
