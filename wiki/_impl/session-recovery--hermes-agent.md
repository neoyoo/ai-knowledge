---
title: "Session Recovery — Hermes Agent"
category: L2
parent: "[[session-recovery]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的 session recovery 以 SQLite 为核心，将会话消息、元数据、推理链全部持久化为结构化关系型存储，并在 resume 时通过 `get_messages_as_conversation()` 重建完整对话历史。其最大设计特色是**压缩触发的 session 分裂**——`_compress_context()` 在压缩上下文时会在 SQLite 里将旧 session 标记为 `ended`、生成新 session ID，并通过 `parent_session_id` 外键将两者链接成链式谱系，使压缩前后的历史可追溯。Gateway 层还提供可配置的 reset policy（idle / daily / both），对多平台（Telegram、Discord、Slack 等）会话实现差异化的生命周期管理。

## 架构分析

### 存储层：单文件 SQLite WAL

`hermes_state.py` 的 `SessionDB` 类是所有持久化操作的入口，数据库路径默认为 `~/.hermes/state.db`（`DEFAULT_DB_PATH = get_hermes_home() / "state.db"`）。

核心表结构（`SCHEMA_SQL`，schema version 6）：

```
sessions
  id, source, user_id, model, model_config, system_prompt
  parent_session_id  ← FOREIGN KEY 自引用，用于链式谱系
  started_at, ended_at, end_reason
  message_count, tool_call_count
  input_tokens, output_tokens, cache_read_tokens, cache_write_tokens, reasoning_tokens
  billing_*, estimated_cost_usd, actual_cost_usd, cost_*
  title

messages
  id (autoincrement), session_id, role, content
  tool_call_id, tool_calls (JSON), tool_name
  timestamp, token_count, finish_reason
  reasoning, reasoning_details (JSON), codex_reasoning_items (JSON)  ← v6 新增
```

v6 迁移增加了 `reasoning / reasoning_details / codex_reasoning_items` 三列，目的是让 OpenRouter、OpenAI、Nous 等重放推理链的 provider 在 session 恢复后能保持多轮推理上下文连续（migration 注释原文："Without these, reasoning chains are lost on session reload, breaking multi-turn reasoning continuity"）。

**并发安全**：多进程（gateway + CLI + worktree agents）共享同一个 `state.db`，使用 WAL 模式 + BEGIN IMMEDIATE + 应用层随机抖动重试（20–150ms，最多 15 次），避免 SQLite 内置确定性退避策略引发的 convoy 效应。每 50 次写操作触发一次 PASSIVE WAL checkpoint 防止文件无限增长。

**FTS5 全文索引**：`messages_fts` 虚拟表通过触发器自动与 `messages` 表同步，支持 `search_messages()` 对所有历史会话做全文检索。

### Session 持久化路径

**CLI 路径**：消息在 `run_agent.py` 的 `_flush_messages_to_session_db()` 中批量写入。该方法用 `_last_flushed_db_idx` 游标追踪已写入位置，多个退出路径（正常结束、工具错误、中断）都会调用，确保只写新增消息不重复。调用链：

```
run_conversation() 内多处
  → _save_session_log(messages)  ← 同时写 ~/.hermes/sessions/session_<id>.json
  → _flush_messages_to_session_db(messages, conversation_history)
      → session_db.ensure_session()  ← INSERT OR IGNORE，容错 startup 时的锁争用
      → session_db.append_message() × N  ← 从 flush_from 索引往后写
```

`_save_session_log()` 另有防护逻辑：若磁盘上已有的 JSON 文件消息数多于当前内存中的消息数，跳过写入——防止 `--resume` 后 agent 持有的部分历史覆盖完整的旧日志。

**Gateway 路径**：`gateway/session.py` 的 `SessionStore.append_to_transcript()` 在 `skip_db=True` 时只写 JSONL 文件（agent 已自行写入 SQLite），否则同时写两处。`rewrite_transcript()` 用于 `/retry`、`/undo`、`/compress` 后原子替换整个 transcript（先 `clear_messages` 再批量 `append_message`，JSONL 文件也同步覆写）。

### Session Resume 流程

**CLI 模式**（`cli.py`，`InteractiveCLI` 类）：

1. `__init__` 时若传入 `resume` 参数，直接将其赋值为 `self.session_id`，并设 `self._resumed = True`
2. `run()` 调用 `_preload_resumed_session()` 提前加载历史供展示（在用户发第一条消息前）：
   - 调用 `session_db.get_session(session_id)` 验证存在性
   - 调用 `session_db.get_messages_as_conversation(session_id)` 获取对话历史，过滤掉 `role == "session_meta"` 的标记消息
   - 调用 `UPDATE sessions SET ended_at = NULL, end_reason = NULL` 重新激活会话
   - 赋值 `self.conversation_history = restored`
3. `_init_agent()` 时若 `self.conversation_history` 非空（已由上一步填充）则跳过 DB 重复查询，直接用 `conversation_history=self.conversation_history` 初始化 `AIAgent`
4. `AIAgent.run_conversation()` 接收 `conversation_history` 并在内部 `messages = list(conversation_history)` 复制

**Gateway 模式**（`gateway/session.py`，`SessionStore` 类）：

`get_or_create_session(source)` 是 gateway 每次收到消息时的入口：
- 计算 session key（`build_session_key(source)`）
- 检查 `_entries` 中是否已有该 key 的 `SessionEntry`
- 若已有，评估 reset policy；未过期则更新 `updated_at` 直接返回
- 若已过期，end 旧 session，创建新 session

消息到来后，gateway 从 SQLite 加载历史传给 agent，agent 完成一轮对话后写回。

**Slack DM Thread 继承**（特殊路径）：当 Slack 创建线程回复时，线程的 session key 与原 DM session key 不同。`get_or_create_session()` 在新建线程 session 时会检测 `source.thread_id` 非空的 DM 情况，自动调用 `load_transcript(parent_entry.session_id)` 并 `rewrite_transcript(entry.session_id, parent_history)` 将父 DM 历史复制过来，避免线程 session 缺少上下文。

### Parent-Child Session 链（压缩触发分裂）

`run_agent.py` 的 `_compress_context()` 是触发点：

```python
# 1. 标记旧 session 结束
session_db.end_session(self.session_id, "compression")
old_session_id = self.session_id

# 2. 生成新 session ID
self.session_id = f"{datetime.now().strftime('%Y%m%d_%H%M%S')}_{uuid.uuid4().hex[:6]}"

# 3. 创建子 session，parent_session_id 指向旧 session
session_db.create_session(
    session_id=self.session_id,
    source=self.platform or ...,
    model=self.model,
    parent_session_id=old_session_id,   # ← 关键链接
)

# 4. 传播 title，自动编号（"my task" → "my task #2"）
new_title = session_db.get_next_title_in_lineage(old_title)
session_db.set_session_title(self.session_id, new_title)

# 5. 重置 flush 游标，新 session 从 0 开始写
self._last_flushed_db_idx = 0
```

`list_sessions_rich()` 默认 `include_children=False`，通过 `WHERE s.parent_session_id IS NULL` 在 UI 中只展示顶级 session，避免压缩链条污染列表视图。

`resolve_session_by_title()` 支持按 title 恢复最新的压缩延续 session："my task" 若已压缩到 "my task #3"，resolve 会返回 #3 的 ID。

### Gateway Session Key 与 SessionSource

`SessionSource` dataclass（`gateway/session.py`）描述消息来源的完整上下文：

```python
@dataclass
class SessionSource:
    platform: Platform       # LOCAL / TELEGRAM / DISCORD / SLACK / WHATSAPP / SIGNAL
    chat_id: str
    chat_name: Optional[str]
    chat_type: str           # "dm" / "group" / "channel" / "thread"
    user_id: Optional[str]
    thread_id: Optional[str]
    user_id_alt: Optional[str]   # Signal UUID（Signal 的 phone/UUID 双 ID 方案）
    chat_id_alt: Optional[str]
```

`build_session_key(source)` 将 SessionSource 映射为确定性字符串 key：
- DM：`agent:main:{platform}:dm:{chat_id}[:{thread_id}]`
- Group/Channel：`agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}][:{user_id}]`

`group_sessions_per_user=True`（默认）：群聊中每个用户独立 session；`thread_sessions_per_user=False`（默认）：线程内所有用户共享一个 session。

### Reset Policy

`SessionResetPolicy`（`gateway/config.py`）支持三种模式：

| mode | 语义 |
|------|------|
| `"none"` | 永不重置，session 长期持续 |
| `"idle"` | 空闲超时（默认 1440 分钟 = 24h）后重置 |
| `"daily"` | 每天固定时间（默认 04:00 本地时间）重置 |
| `"both"` | 空闲或每日时间点，哪个先到触发哪个（默认） |

可按平台（`reset_by_platform`）或会话类型（`reset_by_type`，如 group vs dm）配置不同策略，也可通过环境变量 `SESSION_IDLE_MINUTES` 和 `SESSION_RESET_HOUR` 动态覆盖。

Session 过期时，新的 `SessionEntry` 会携带 `was_auto_reset=True` 和 `auto_reset_reason`（"idle"/"daily"），由 message handler 在下一条消息时注入通知给用户。

### 消息重建：`get_messages_as_conversation()`

`hermes_state.py` 中该方法将 SQLite 行重建为 OpenAI messages 格式：

```python
SELECT role, content, tool_call_id, tool_calls, tool_name,
       reasoning, reasoning_details, codex_reasoning_items
FROM messages WHERE session_id = ? ORDER BY timestamp, id
```

重建时：
- `tool_calls` JSON 字符串反序列化为 list
- assistant 消息附加 `reasoning` / `reasoning_details` / `codex_reasoning_items`（v6 新增），确保推理型 provider 恢复后能正确继续推理链
- `role == "session_meta"` 的标记行在 CLI 恢复时被过滤掉

### Shadow-Git 文件系统检查点

`tools/checkpoint_manager.py` 实现了一套对 LLM 完全透明的文件系统快照系统，与 SQLite 会话恢复形成正交的双重保护层——前者恢复对话历史，后者恢复磁盘文件状态。

**Shadow Git Repo 的创建方式**

每个工作目录对应一个独立的 shadow git repo，路径为 `~/.hermes/checkpoints/{sha256(abs_path)[:16]}/`，用工作目录绝对路径的 SHA-256 前 16 位做目录名，保证确定性映射。初始化时通过 `GIT_DIR=<shadow_repo> GIT_WORK_TREE=<project_dir>` 环境变量将 git 操作重定向到 shadow 目录，**项目自身的 `.git/` 完全不受影响**。`_init_shadow_repo()` 在 shadow repo 内写入 `info/exclude`（屏蔽 `node_modules/`、`.env`、`.venv/` 等 17 类噪声路径）和 `HERMES_WORKDIR` 文件（记录对应的原始工作目录路径）。

**每轮写操作前的快照机制**

`CheckpointManager` 由 `AIAgent` 持有，在对话循环的每个 turn 开始时调用 `new_turn()` 清空 `_checkpointed_dirs` 集合（per-turn 去重标记）。当 `write_file`、`patch` 等文件变更工具被调用前，agent 调用 `ensure_checkpoint(working_dir, reason)`：

1. 检查 `enabled` 开关（由 `--checkpoints` CLI flag 或 config 控制）
2. 惰性探测 `git` 可执行文件是否存在
3. 若该目录本轮已快照则直接跳过（同一 turn 内一个目录最多拍一次）
4. 调用 `_take()` 执行实际快照：`git add -A` → 检查 `--cached` diff 是否有变化 → `git commit -m <reason>`

整个过程封装在 try/except 中，快照失败不会阻断工具执行（`ensure_checkpoint` 永不抛异常）。`_MAX_FILES = 50,000` 防止对超大目录执行耗时快照。

**检查点存储位置与上限**

所有 shadow repo 统一存放在 `~/.hermes/checkpoints/`（由 `get_hermes_home()` 决定，与 `state.db` 同根）。每个 shadow repo 默认保留最近 50 条提交（`max_snapshots=50`）。`_prune()` 方法不做实际的 git history 裁剪（rebase/filter-branch 对后台特性过于脆弱），而是靠 `list_checkpoints()` 在展示时用 `-n max_snapshots` 限制日志条数。

**回滚能力**

`restore(working_dir, commit_hash, file_path=None)` 通过 `git checkout <hash> -- .`（或单文件）将文件状态还原到指定检查点，**不移动 HEAD**（安全且可逆）。还原前会先调用 `_take()` 保存当前状态（"pre-rollback snapshot"），使"撤销回滚"也成为可能。`diff(working_dir, commit_hash)` 支持预览检查点与当前工作树之间的差异，结合 `list_checkpoints()` 提供完整的检查点浏览体验。

**设计动机：不污染项目 git**

shadow git 架构的核心考量是**恢复能力与项目 git 历史解耦**：agent 的每步写操作快照不会产生"hermes checkpoint"类型的垃圾提交出现在 `git log` 里，用户的项目历史保持干净。与此同时，`get_working_dir_for_path()` 通过向上查找 `.git`、`pyproject.toml`、`package.json` 等 marker 自动定位项目根，快照粒度是项目级而非文件级，一次 commit 捕获整个工作树的变更快照，与 git 的语义保持一致。

### 容错机制

**Session 创建失败容错**：`_flush_messages_to_session_db()` 在写消息前调用 `session_db.ensure_session()`（`INSERT OR IGNORE`），以应对 agent 启动时因 SQLite 锁争用导致 `create_session()` 失败的情况——session 行可能不存在，但消息仍然能被写入。

**压缩分裂失败容错**：`_compress_context()` 的 SQLite 操作套在 try/except 中，失败时记录 warning 但压缩本身继续（压缩后的消息仍在内存中，但新 session 不会被索引）。

**Session log 防覆盖**：`_save_session_log()` 读取磁盘上已有 JSON 的消息数，若磁盘记录更多则跳过写入，防止 resume 时 agent 的初始内存状态覆盖更完整的历史文件。

**Crash 时的容错能力**：Hermes **没有**每轮自动写入 SQLite 的 checkpoint 机制——`_flush_messages_to_session_db()` 是在 `run_conversation()` 内多个退出点调用，但若进程在 tool 执行中途被 kill（SIGKILL），当轮消息可能丢失。`_save_session_log()` 同样是轮末保存，JSON 文件也不能防止中途崩溃丢失当前轮。

## 关键代码路径

| 文件 | 类/函数 | 作用 |
|------|---------|------|
| `hermes_state.py` | `SessionDB.create_session()` | 创建 session 行，接受 parent_session_id |
| `hermes_state.py` | `SessionDB.end_session()` | 标记 ended_at + end_reason |
| `hermes_state.py` | `SessionDB.reopen_session()` | 清除 ended_at，激活已结束的 session |
| `hermes_state.py` | `SessionDB.get_messages_as_conversation()` | 从 messages 表重建 OpenAI 消息格式 |
| `hermes_state.py` | `SessionDB.search_messages()` | FTS5 全文检索，附带 snippet 和上下文 |
| `hermes_state.py` | `SessionDB.resolve_session_by_title()` | 按 title 找最新的压缩延续 session |
| `hermes_state.py` | `SessionDB.get_next_title_in_lineage()` | 生成 "name #N" 自动编号 title |
| `hermes_state.py` | `SessionDB._execute_write()` | 带随机抖动重试的写事务包装器 |
| `run_agent.py` | `AIAgent._flush_messages_to_session_db()` | 批量写新消息，`_last_flushed_db_idx` 防重复 |
| `run_agent.py` | `AIAgent._compress_context()` | 压缩 + session 分裂 + parent_session_id 链接 |
| `run_agent.py` | `AIAgent._save_session_log()` | 写 JSON 全量日志，有防覆盖保护 |
| `run_agent.py` | `AIAgent.flush_memories()` | 压缩前触发一轮 memory 工具调用，防止记忆丢失 |
| `cli.py` | `InteractiveCLI._preload_resumed_session()` | 早期加载历史供展示，在第一次用户输入前 |
| `cli.py` | `InteractiveCLI._init_agent()` | 验证 session 存在性，初始化带历史的 AIAgent |
| `gateway/session.py` | `SessionStore.get_or_create_session()` | gateway 会话入口，评估 reset policy |
| `gateway/session.py` | `SessionStore.switch_session()` | `/resume` 命令：将 session key 重定向到指定历史 session |
| `gateway/session.py` | `build_session_key()` | 从 SessionSource 生成确定性 session key |
| `gateway/config.py` | `SessionResetPolicy` | 可配置的 reset 策略（mode/at_hour/idle_minutes） |
| `tools/checkpoint_manager.py` | `CheckpointManager.ensure_checkpoint()` | 写工具调用前的 per-turn 去重快照入口，永不抛异常 |
| `tools/checkpoint_manager.py` | `CheckpointManager._take()` | 实际执行 shadow git add + commit，跳过无变更目录 |
| `tools/checkpoint_manager.py` | `CheckpointManager.restore()` | 按 commit hash 回滚文件，回滚前自动保存当前快照 |
| `tools/checkpoint_manager.py` | `CheckpointManager.list_checkpoints()` | 列出某目录的历史快照（含 diffstat），最新在前 |
| `tools/checkpoint_manager.py` | `_shadow_repo_path()` | 将工作目录路径 hash 映射到 ~/.hermes/checkpoints/<hash[:16]>/ |
| `tools/checkpoint_manager.py` | `_git_env()` | 构造 GIT_DIR + GIT_WORK_TREE 环境变量，隔离 shadow repo |

## 设计亮点

**压缩谱系（parent_session_id 链）**：压缩不是破坏性操作，而是产生一个有父子关系的新 session。`list_sessions_rich()` 默认隐藏子 session，`resolve_session_by_title()` 自动找到最新的延续 session，用户体验上一个 "project" 可以无缝跨越多次压缩。

**推理链持久化（v6）**：`reasoning / reasoning_details / codex_reasoning_items` 三列的引入直接解决了"重放推理"型 provider（OpenRouter、Nous）在 session 恢复后推理上下文中断的问题，属于对 provider 行为差异的精确适配。

**FTS5 全文召回**：所有会话消息自动进入 `messages_fts` 虚拟表，支持布尔运算、前缀匹配、短语搜索；`search_messages()` 返回包含 snippet 和前后各一条消息的上下文，agent 可用此工具检索历史决策依据。

**双轨持久化**：SQLite 作为主存储，JSONL 文件作为兼容性辅助存储并存，`append_to_transcript(skip_db=True)` 避免 agent 自写 + gateway 写的双重写入 bug，过渡设计的工程细节处理干净。

**PII 安全的 session key**：`_hash_sender_id()` / `_hash_chat_id()` 对 WhatsApp / Signal / Telegram 等平台的 user_id 做 SHA-256 截断哈希，系统 prompt 注入的是哈希值，路由层保留原始值，PII 不进入 LLM context。

**title 谱系自动编号**：`get_next_title_in_lineage()` 扫描所有 "name #N" 变体取最大值 +1，`/title` 命令给当前 session 命名后，压缩延续 session 自动继承 "name #2" / "name #3"，避免用户手动管理命名。

**memory flush 前置于压缩**：`_compress_context()` 调用 `flush_memories(messages, min_turns=0)` 让模型在压缩丢弃中间轮次前先调用 memory 工具保存关键信息，之后的 `_generate_summary()` 再产出结构化摘要（Goal/Progress/Decisions/Files/Next Steps），形成双层记忆保护。

**Shadow-Git 透明文件检查点**：`CheckpointManager` 在每个写操作工具（`write_file`、`patch`）执行前，以 per-turn 去重的方式向 `~/.hermes/checkpoints/` 下的 shadow git repo 提交一次快照，使用 `GIT_DIR + GIT_WORK_TREE` 完全绕开项目自身的 `.git/`——agent 的操作记录不污染用户的 git 历史。`restore()` 通过 `git checkout <hash> -- .` 实现可逆回滚，且回滚前自动保存当前状态（支持"撤销回滚"），将文件级容错能力从"轮末"细化到"写操作前"粒度。

## 局限性

**无增量 checkpoint，崩溃会丢失当轮消息**：`_flush_messages_to_session_db()` 和 `_save_session_log()` 均为轮末调用，若进程在 tool 执行中途被 kill，当前轮所有消息（包括已执行的 tool call/result）将丢失。Claude Code 通过 transcript 文件的每步追加写入解决了这个问题，Hermes 没有类似机制。

**Gateway 历史加载路径依赖 JSONL 文件**：`SessionStore.load_transcript()` 和 `rewrite_transcript()` 的实际实现（文件读取部分）依赖 legacy JSONL 文件，SQLite 是主存储但 gateway 在 `/compress` 等场景下仍通过文件操作管理 transcript，双轨并存增加了一致性维护成本。

**session_meta 行处理依赖过滤**：`get_messages_as_conversation()` 返回所有 messages 包括 `role="session_meta"` 的特殊标记行，CLI 恢复时需要手动过滤（`[m for m in restored if m.get("role") != "session_meta"]`），不是干净的分层设计。

**Reset policy 在进程内存中评估，无持久化**：`_should_reset()` 基于 `SessionEntry.updated_at`（存在 `sessions.json` 中），若 gateway 进程意外重启，`_entries` 需要从 `sessions.json` 重新加载，reset policy 的状态依赖文件而非 SQLite，在极端情况（sessions.json 损坏）下可能失效。

**压缩分裂失败不回滚**：`_compress_context()` 内 SQLite 操作失败时，新 session 不会被创建（旧 session 已被 `end_session` 标记结束），消息写入会继续使用旧 session ID——因为 `_last_flushed_db_idx = 0` 已被重置，后续 flush 会尝试向已结束的旧 session 写入，可能导致消息计入错误 session。

## 来源

- 源码版本：hermes-agent 0.16.0
- 分析深度：源码级
- 主要文件：`hermes_state.py`（1304 行）、`run_agent.py`（9431 行）、`cli.py`（8736 行）、`gateway/session.py`、`gateway/config.py`、`agent/context_compressor.py`
