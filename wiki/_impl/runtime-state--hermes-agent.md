---
title: "Runtime State — Hermes Agent"
category: L2
parent: "[[runtime-state]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的运行时状态系统以 `hermes_state.py` 中的 `SessionDB` 类为核心，以 SQLite 文件 `~/.hermes/state.db` 作为唯一的持久化后端。它取代了早期"每次会话写一个 JSONL 文件"的方案，统一管理 session 元数据、完整消息历史、token 用量和费用核算。`SessionDB` 的设计定位是"多进程共享的会话数据库"——gateway（消息通道）、CLI、worktree agent 可以同时连接同一个文件，依靠 WAL 模式和应用层抖动重试处理并发写入竞争。

## 架构分析

### SQLite 状态存储 — WAL + FTS5 双轨

`SessionDB`（`hermes_state.py:114`）在初始化时做三件事：

1. 以 `isolation_level=None`（手动事务）和 `timeout=1.0` 打开连接，关闭 Python 默认的自动事务模式，防止与显式 `BEGIN IMMEDIATE` 冲突；
2. 执行 `PRAGMA journal_mode=WAL` 和 `PRAGMA foreign_keys=ON`；
3. 调用 `_init_schema()` 完成表结构创建和 schema 版本迁移。

WAL 模式允许多个读者和一个写者并行访问，是网关多平台并发场景的核心前提。FTS5 虚拟表通过触发器与 `messages` 表保持同步，支持跨所有历史消息的全文检索。

### Schema 设计 — 三张表

**sessions 表**（`SCHEMA_SQL`，`hermes_state.py:40`）：

| 字段 | 类型 | 用途 |
|-----|------|------|
| `id` | TEXT PK | 时间戳+短 UUID 格式：`20250408_143022_a3f1b2` |
| `source` | TEXT NOT NULL | 来源标签：`cli`、`telegram`、`discord`、`gateway` 等 |
| `parent_session_id` | TEXT FK | 压缩分裂时指向上一个 session |
| `started_at` / `ended_at` | REAL | Unix 时间戳 |
| `end_reason` | TEXT | 结束原因：`compression`、`user_exit` 等 |
| `message_count` / `tool_call_count` | INTEGER | 计数器，每次 `append_message` 原子递增 |
| `input_tokens` / `output_tokens` / `cache_read_tokens` / `cache_write_tokens` / `reasoning_tokens` | INTEGER | 五维 token 用量（v5 迁移加入） |
| `billing_provider` / `billing_base_url` / `billing_mode` | TEXT | 计费路由三元组 |
| `estimated_cost_usd` / `actual_cost_usd` / `cost_status` / `cost_source` / `pricing_version` | — | 费用核算五元组 |
| `title` | TEXT | 人类可读标题（带唯一约束，NULL 允许多条） |
| `model` / `model_config` / `system_prompt` / `user_id` | — | 模型和提示词快照 |

**messages 表**（`SCHEMA_SQL`，`hermes_state.py:70`）：

| 字段 | 用途 |
|-----|------|
| `role` | `user` / `assistant` / `tool` / `system` |
| `content` | 消息正文 |
| `tool_call_id` / `tool_calls` / `tool_name` | 工具调用三件套，`tool_calls` 序列化为 JSON |
| `finish_reason` | `stop`、`tool_calls`、`length` 等 |
| `reasoning` / `reasoning_details` / `codex_reasoning_items` | v6 新增：推理链字段，支持 OpenRouter/OpenAI/Nous 的多轮推理重放 |
| `token_count` | 可选的单条消息 token 估计 |

**messages_fts 虚拟表**（`FTS_SQL`，`hermes_state.py:92`）：

- `USING fts5(content, content=messages, content_rowid=id)` — 内容表模式，不额外存储内容
- 三个触发器（INSERT / DELETE / UPDATE on messages）保持 FTS 索引同步
- 查询时对用户输入做六步清洗（`_sanitize_fts5_query`），处理 FTS5 特殊字符、布尔运算符、连字符词组

### Schema 版本迁移 — SCHEMA_VERSION = 6

`_init_schema()`（`hermes_state.py:252`）采用线性升级策略：

- v1 → v2：messages 加 `finish_reason`
- v2 → v3：sessions 加 `title`
- v3 → v4：创建 `title` 字段的条件唯一索引（`WHERE title IS NOT NULL`）
- v4 → v5：sessions 加五个 token 字段 + 八个计费/费用字段（11 列一次迁移）
- v5 → v6：messages 加 `reasoning` / `reasoning_details` / `codex_reasoning_items`（保存推理链，防止多轮推理丢失）

每个版本升级用 `ALTER TABLE ... ADD COLUMN` 实现，捕获 `OperationalError`（列已存在时静默忽略），升级后更新 `schema_version` 表。FTS5 虚拟表单独创建，不放入 `executescript`（避免 SQLite 的 `IF NOT EXISTS` 兼容性问题）。

### 并发写入控制 — 抖动重试 + 被动 WAL 检查点

`_execute_write()`（`hermes_state.py:163`）是所有写操作的统一入口：

- 每次写操作使用 `BEGIN IMMEDIATE` 事务，在事务开始时立即抢占 WAL 写锁，而不是在 commit 时才竞争
- `timeout=1.0`（SQLite 层）配合应用层最多 15 次重试（`_WRITE_MAX_RETRIES = 15`）
- 每次重试前随机睡眠 20-150ms（`_WRITE_RETRY_MIN_S = 0.020`，`_WRITE_RETRY_MAX_S = 0.150`），打破 SQLite 内置 busy handler 的确定性退避造成的"护送效应"（convoy effect）
- 每写入 50 次（`_CHECKPOINT_EVERY_N_WRITES = 50`）触发一次 `PRAGMA wal_checkpoint(PASSIVE)`，被动将已提交的 WAL 帧刷回主数据库文件，防止 WAL 文件无限增长

读操作直接用 `self._lock`（threading.Lock）保护，不走重试路径。

### 费用核算 — `agent/usage_pricing.py`

费用核算系统由四个类型化数据类构成：

- `CanonicalUsage`：五维 token 桶（input / output / cache_read / cache_write / reasoning）
- `BillingRoute`：计费路由（provider + model + base_url + billing_mode），由 `resolve_billing_route()` 推断
- `PricingEntry`：每百万 token 价格 + 来源元数据（`CostSource` 枚举）
- `CostResult`：计算结果，携带 `CostStatus`（`actual` / `estimated` / `included` / `unknown`）

`normalize_usage()` 处理三种 API 形态的 token 字段归一化：
- Anthropic：直接读 `cache_read_input_tokens` / `cache_creation_input_tokens`
- OpenAI Chat Completions：`prompt_tokens_details.cached_tokens` 减法推导
- Codex Responses：`input_tokens_details.cached_tokens` 减法推导

静态定价表 `_OFFICIAL_DOCS_PRICING` 内置 Anthropic / OpenAI / DeepSeek / Google 主流模型的官方价格，OpenRouter 通过 Models API 动态拉取。

### Session 生命周期

**创建**（`run_agent.py:901`）：

```python
timestamp_str = self.session_start.strftime("%Y%m%d_%H%M%S")
short_uuid = uuid.uuid4().hex[:6]
self.session_id = f"{timestamp_str}_{short_uuid}"
```

格式示例：`20250408_143022_a3f1b2`。如果调用方传入了 `session_id`（CLI 从配置注入），则使用传入值。

`AIAgent.__init__` 随即调用 `session_db.create_session()`，失败时（SQLite 锁竞争）记录 WARNING 但不中断 — `_session_db` 保持有效，后续消息刷写时通过 `ensure_session()` 的 `INSERT OR IGNORE` 恢复。

**消息刷写**（`run_agent.py:1855`）：

`_flush_messages_to_session_db()` 用 `_last_flushed_db_idx` 游标追踪已写入位置，每次调用只写入新增消息，防止重复写入（fix #860）。`_persist_session()` 在所有退出路径（正常完成、中断、异常、压缩）均会被调用，确保消息不丢失。

**Token 计数更新**（`run_agent.py:7746`）：

每次 API 调用完成后，立即调用 `session_db.update_token_counts()`，传入五维 canonical usage + 费用结果 + 计费路由信息。CLI 路径使用增量模式（`absolute=False`，累加），Gateway 路径使用绝对模式（`absolute=True`，覆盖），避免 gateway 每条消息创建新 agent 实例导致的重复计数。

**压缩分裂**（`run_agent.py:5867`）：

context 压缩触发时，当前 session 以 `end_reason="compression"` 结束，新建一个 session 并将旧 session id 设为 `parent_session_id`，形成压缩链。title 通过 `get_next_title_in_lineage()` 自动续号（`"my session"` → `"my session #2"`）。

### 标题系统

`auto_title_session()`（`agent/title_generator.py:59`）在首次响应后由守护线程异步触发：

1. 仅在对话前两轮触发（`user_msg_count <= 2`）
2. 调用辅助 LLM（最便宜/最快模型，`task="compression"` 配置）生成 3-7 词标题，`max_tokens=30`，`temperature=0.3`
3. 标题唯一性由 `SET UNIQUE INDEX ... WHERE title IS NOT NULL` 保证，冲突抛 `ValueError`
4. `sanitize_title()` 清除 ASCII 控制字符、零宽字符、RTL/LTR 覆盖码，并折叠多余空白，最大 100 字符

### 凭证池 — `agent/credential_pool.py`

`CredentialPool` 类（`credential_pool.py:352`）为同一 Provider 维护多个 API Key / OAuth Token，实现无感知失败切换。核心设计如下：

**四种轮换策略**（`SUPPORTED_POOL_STRATEGIES`，`credential_pool.py:58`）：

| 策略 | 行为 |
|------|------|
| `fill_first` | 默认策略，始终优先使用 priority 最低的凭证，直到它被耗尽才换下一个 |
| `round_robin` | 每次选取第一个可用凭证后，将其移到优先级末位，并重新编号，实现公平轮转 |
| `random` | 从当前可用凭证中随机选取，打散请求分布 |
| `least_used` | 选取 `request_count` 最小的凭证，追踪总调用次数，配合 `mark_used()` 工作 |

策略在 `config.yaml` 的 `credential_pool_strategies.<provider>` 字段配置，`get_pool_strategy()` 读取，非法值静默回落到 `fill_first`。

**按错误码区分冷却时长**（`credential_pool.py:68`）：

```python
EXHAUSTED_TTL_429_SECONDS = 60 * 60          # 1 小时（限流，配额通常快速重置）
EXHAUSTED_TTL_DEFAULT_SECONDS = 24 * 60 * 60 # 24 小时（402 欠费/配额耗尽等）
```

`_exhausted_ttl(error_code)` 根据 HTTP 状态码返回冷却秒数；`_exhausted_until()` 还会优先解析 Provider 返回的 `reset_at` / `resets_at` / `retry_until` 字段（支持 epoch 秒、毫秒、ISO-8601），并从错误消息中提取 `quotaResetDelay` / `retry after N sec` 等文本，以实际重置时间为准，避免无谓等待。

**两种认证类型**（`credential_pool.py:49`）：

- `auth_type = "oauth"`：持有 `access_token` + `refresh_token`，过期前自动调用 `_refresh_entry()` 续期，支持 Anthropic PKCE、OpenAI Codex、Nous 三套 OAuth 刷新路径；
- `auth_type = "api_key"`：直接使用 `access_token` 字段，无需刷新。

OAuth 凭证还有**跨进程单次消费令牌同步**机制：Anthropic OAuth 刷新令牌是单次使用的，如果 Claude Code CLI 或另一个 Profile 先刷新了，当前 Pool 条目会失效。`_sync_anthropic_entry_from_credentials_file()` 在凭证被标记 exhausted 时检查 `~/.claude/.credentials.json`，若发现令牌已被更新则直接同步，不需要重新请求 Auth Server。

**线程安全**：`CredentialPool._lock`（`threading.Lock`）保护 `select()`、`mark_exhausted_and_rotate()`、`mark_used()` 等所有修改内部状态的操作（`credential_pool.py:358`）。同时提供软租约系统（`acquire_lease` / `release_lease`），按 `_max_concurrent`（默认 1）限制同一凭证的并发使用数。

**`custom:` 前缀用于 OpenAI 兼容端点**（`credential_pool.py:75`）：

凡是在 `config.yaml.custom_providers` 列表中定义的自定义端点，其 Pool Key 均以 `custom:` 为前缀（如 `custom:together-ai`），由 `get_custom_provider_pool_key(base_url)` 根据 URL 匹配并生成，与官方 Provider 的 Pool Key 严格隔离。

### 多 Profile 隔离 — `hermes_cli/profiles.py`

Profile 系统（`profiles.py`）允许在同一台机器上运行多个完全隔离的 Hermes 实例，每个实例拥有独立的 `config.yaml`、`.env`、凭证池、内存、skills、gateway 和日志。

**目录结构**：

- 默认 Profile：`~/.hermes`（零迁移成本，向后兼容）
- 命名 Profile：`~/.hermes/profiles/<name>/`（如 `~/.hermes/profiles/coder/`）
- Profile 根目录固定为 `~/.hermes/profiles/`，锚定到用户 home，而非当前 `HERMES_HOME`，确保 `hermes profile list` 能看到所有 Profile（`profiles.py:116`）

每个命名 Profile 目录包含 `memories/`、`sessions/`、`skills/`、`skins/`、`logs/`、`plans/`、`workspace/`、`cron/` 八个子目录，创建时自动 bootstrap。

**`--clone` 和 `--clone-all` 旗标**（`create_profile()`，`profiles.py:366`）：

| 旗标 | 复制内容 |
|------|---------|
| 无旗标 | 仅 bootstrap 空目录结构 |
| `--clone` | 复制 `config.yaml`、`.env`、`SOUL.md`、`memories/MEMORY.md`、`memories/USER.md`（身份延续） |
| `--clone-all` | `shutil.copytree` 完整拷贝，再删除 `gateway.pid`、`gateway_state.json`、`processes.json` 等运行时文件 |

`--clone` / `--clone-all` 的克隆源默认为当前活跃 Profile；可通过 `--clone-from <name>` 指定。

**Wrapper 别名**（`create_wrapper_script()`，`profiles.py:212`）：

创建 Profile 时默认在 `~/.local/bin/<name>` 写入一行 shell 脚本：

```sh
#!/bin/sh
exec hermes -p <name> "$@"
```

脚本可执行位设置为 `u+x,g+x,o+x`。删除 Profile 时自动检查并移除（仅删除 Hermes 生成的 wrapper，避免误删同名系统命令）。`_is_wrapper_dir_in_path()` 检测 `~/.local/bin` 是否在 `PATH` 中并给出提示。

**Profile 切换如何影响运行时 `HERMES_HOME`**（`resolve_profile_env()`，`profiles.py:1054`）：

CLI 入口点解析 `-p <name>` 时，在任何 Hermes 模块被 import 之前调用 `resolve_profile_env(profile_name)`，将返回的路径字符串设置为 `HERMES_HOME` 环境变量。因为 `hermes_constants.get_hermes_home()` 读取 `os.getenv("HERMES_HOME")`，所有后续模块加载都会以该 Profile 目录为根目录，相当于隔离沙箱。粘性默认 Profile 存储在 `~/.hermes/active_profile` 文件（原子写 `.tmp` → rename）。`get_active_profile_name()` 通过比较当前 `HERMES_HOME` 与 `~/.hermes/profiles/` 路径关系来反向推断当前 Profile 名称（`profiles.py:700`）。

### 日志脱敏 — `agent/redact.py`

`redact.py` 在所有日志落盘路径上运行 `RedactingFormatter`，防止 API Key、OAuth Token、私钥等敏感字符串写入磁盘。

**30+ 前缀模式正则**（`_PREFIX_PATTERNS`，`redact.py:20`）：

`_PREFIX_PATTERNS` 列表包含 34 个具名 API Key 前缀的正则片段，覆盖 OpenAI/Anthropic（`sk-`）、GitHub（`ghp_`/`gho_`/`ghu_`/`ghs_`/`ghr_`/`github_pat_`）、Slack（`xox[baprs]-`）、Google（`AIza`）、AWS（`AKIA`）、Stripe（`sk_live_`/`sk_test_`/`rk_live_`）、HuggingFace（`hf_`）、Groq（`gsk_`）、Tavily（`tvly-`）等主流服务的 Token 格式。所有片段在 import 时合并为单个交替正则 `_PREFIX_RE`，带边界断言（`(?<![A-Za-z0-9_-])` / `(?![A-Za-z0-9_-])`）防止误匹配路径或标识符。

**ENV 赋值模式检测**（`_ENV_ASSIGN_RE`，`redact.py:60`）：

独立正则匹配形如 `OPENAI_API_KEY=sk-abc` 的字符串——要求变量名含 `API_KEY`、`TOKEN`、`SECRET`、`PASSWORD`、`CREDENTIAL`、`AUTH` 等关键词（大小写不限），捕获值部分后独立脱敏。此规则覆盖了 `.env` 文件内容被打印到日志、或 LLM 生成 `export` 命令被记录的场景。

**import 时快照 `HERMES_REDACT_SECRETS`**（`redact.py:17`）：

```python
_REDACT_ENABLED = os.getenv("HERMES_REDACT_SECRETS", "").lower() not in ("0", "false", "no", "off")
```

在模块 import 时立即读取并缓存为模块级变量，之后运行时对该环境变量的修改（包括 LLM 生成的 `export HERMES_REDACT_SECRETS=false`）无法禁用脱敏，防止会话期间被攻击者或失控代码绕过。

**部分保留 Token 用于调试**（`_mask_token()`，`redact.py:106`）：

```python
def _mask_token(token: str) -> str:
    if len(token) < 18:
        return "***"           # 短 Token 完全遮盖
    return f"{token[:6]}...{token[-4:]}"  # 长 Token 保留前 6 + 后 4
```

保留头尾字符让工程师能在日志中区分同一 Provider 的多个 Key（例如 `sk-ant-...xyz1` 与 `sk-ant-...xyz2`），同时脱敏中间敏感部分。

除前缀模式外，`redact_sensitive_text()` 还串行处理：JSON 字段（`"apiKey": "..."`）、Authorization header（`Bearer <token>`）、Telegram Bot Token（`bot<digits>:<token>`）、PEM 私钥块、数据库连接串密码（`postgres://user:PASSWORD@host`）和 E.164 电话号码（`+<country><number>`）。

### 配置状态 — `hermes_constants.py` + `hermes_cli/config.py`

**HERMES_HOME 解析**（`hermes_constants.py:11`）：

```python
def get_hermes_home() -> Path:
    return Path(os.getenv("HERMES_HOME", Path.home() / ".hermes"))
```

所有路径（state.db、config.yaml、logs/、sessions/、memories/）都从这个单一入口派生，环境变量优先。`get_hermes_dir()` 提供带向后兼容的子目录解析：旧路径存在则用旧路径，否则用新路径。

**config.yaml 加载**（`hermes_cli/config.py:201`）：

`DEFAULT_CONFIG` 字典定义所有默认值，`load_config()` 读取 `~/.hermes/config.yaml` 后深度合并覆盖。关键配置域：
- `agent.max_turns`（默认 90）、`agent.gateway_timeout`（默认 1800s）
- `compression.threshold`（0.50）、`compression.target_ratio`（0.20）、`compression.protect_last_n`（20）
- `terminal.backend`（`local`/`ssh`/`docker`/`modal`）、`terminal.timeout`（180s）

**日志系统**（`hermes_logging.py`）：

`setup_logging()` 配置两个 RotatingFileHandler：
- `~/.hermes/logs/agent.log`（INFO+，默认 5MB × 3 备份）
- `~/.hermes/logs/errors.log`（WARNING+，2MB × 2 备份）

两者均使用 `RedactingFormatter`（`agent/redact.py`），在落盘前自动脱敏 API Key 等敏感字段。`--verbose` 模式通过 `setup_verbose_logging()` 额外附加 `StreamHandler`（DEBUG 级别）。

**时区**（`hermes_time.py`）：

`now()` 返回时区感知的 datetime，解析优先级：`HERMES_TIMEZONE` 环境变量 → `config.yaml.timezone` → 服务器本地时区。结果缓存，`reset_cache()` 用于测试和配置变更后的强制重解析。

## 关键代码路径

- `hermes_state.py:114` — `SessionDB` 类，SQLite 状态存储核心
- `hermes_state.py:163` — `_execute_write()`，写事务抖动重试入口
- `hermes_state.py:216` — `_try_wal_checkpoint()`，被动 WAL 检查点
- `hermes_state.py:252` — `_init_schema()`，Schema 创建和线性版本迁移
- `hermes_state.py:355` — `create_session()` / `end_session()` / `reopen_session()`，session 生命周期
- `hermes_state.py:412` — `update_token_counts()`，增量/绝对两种 token 累加模式
- `hermes_state.py:857` — `append_message()`，消息写入 + 计数器原子更新
- `hermes_state.py:951` — `get_messages_as_conversation()`，还原 OpenAI 格式对话历史（含推理链）
- `hermes_state.py:999` — `_sanitize_fts5_query()` + `search_messages()`，FTS5 全文检索
- `hermes_state.py:672` — `set_session_title()` / `sanitize_title()` / `get_next_title_in_lineage()`，标题管理
- `run_agent.py:901` — `AIAgent.__init__` 中 session_id 生成逻辑
- `run_agent.py:1855` — `_flush_messages_to_session_db()`，增量游标刷写，防重复
- `run_agent.py:5867` — 压缩触发的 session 分裂 + parent_session_id 链构建
- `run_agent.py:7746` — API 调用后 token 计数实时更新
- `agent/usage_pricing.py:420` — `normalize_usage()`，三种 API 形态 token 字段归一化
- `agent/usage_pricing.py:481` — `estimate_usage_cost()`，五维 token 费用估算
- `agent/title_generator.py:95` — `maybe_auto_title()`，首次响应后异步标题生成
- `hermes_constants.py:11` — `get_hermes_home()`，HERMES_HOME 单一解析入口
- `hermes_cli/config.py:185` — `ensure_hermes_home()`，目录结构初始化（0700 权限）
- `hermes_logging.py:50` — `setup_logging()`，双文件 RotatingFileHandler + 脱敏格式化
- `agent/credential_pool.py:352` — `CredentialPool` 类，多凭证轮换池核心
- `agent/credential_pool.py:333` — `get_pool_strategy()`，读取 config.yaml 策略配置
- `agent/credential_pool.py:706` — `_select_unlocked()`，四策略分发入口（fill_first / round_robin / random / least_used）
- `agent/credential_pool.py:187` — `_exhausted_ttl()`，按 HTTP 状态码区分冷却时长（429 = 1h，其他 = 24h）
- `agent/credential_pool.py:262` — `_exhausted_until()`，解析 Provider reset 时间戳或退避延迟
- `agent/credential_pool.py:410` — `_sync_anthropic_entry_from_credentials_file()`，跨进程 OAuth 令牌同步
- `agent/credential_pool.py:743` — `mark_exhausted_and_rotate()`，标记耗尽并立即切换下一个凭证
- `hermes_cli/profiles.py:366` — `create_profile()`，含 --clone / --clone-all 逻辑
- `hermes_cli/profiles.py:212` — `create_wrapper_script()`，写入 `~/.local/bin/<name>` shell 别名
- `hermes_cli/profiles.py:1054` — `resolve_profile_env()`，CLI 入口点 import 前设置 HERMES_HOME
- `hermes_cli/profiles.py:700` — `get_active_profile_name()`，从 HERMES_HOME 反推当前 Profile 名
- `hermes_cli/profiles.py:116` — `_get_profiles_root()`，锚定到 home，非当前 HERMES_HOME
- `agent/redact.py:17` — `_REDACT_ENABLED`，import 时快照，防止运行时被绕过
- `agent/redact.py:20` — `_PREFIX_PATTERNS`，34 条命名 API Key 前缀正则
- `agent/redact.py:59` — `_ENV_ASSIGN_RE`，ENV 赋值模式检测（`KEY=value`）
- `agent/redact.py:106` — `_mask_token()`，短 Token 完全遮盖，长 Token 保留前 6 + 后 4
- `agent/redact.py:113` — `redact_sensitive_text()`，串行执行八类脱敏规则的主函数

## 设计亮点

**应用层抖动重试彻底替代 SQLite 内置 busy handler**：SQLite 内置的 busy handler 使用确定性退避，多进程同时等待时容易形成护送效应（convoy effect）——所有进程以相同节奏碰撞、等待、碰撞。Hermes 将 SQLite timeout 设为 1s，在应用层用随机 20-150ms 抖动重试（最多 15 次），自然错开竞争窗口。配合 `BEGIN IMMEDIATE`（事务开始立即抢锁，不等到 commit），锁竞争会尽早暴露，不会在 commit 时突然失败。

**增量游标防重复写入**：`_last_flushed_db_idx` 游标记录上次刷写位置，多个退出路径（正常结束、中断、异常、压缩）都调用同一个 `_persist_session()`，不会因为多次调用产生重复消息行。这是 fix #860 的关键。

**压缩触发的 session 链式分裂**：context 压缩时不覆盖原 session，而是 `end_session(old_id, "compression")` + `create_session(new_id, parent_session_id=old_id)`，形成可追溯的压缩链。title 自动续号（`#2`、`#3`……）让用户能在列表里找到所有相关分段。

**五维 token + 完整费用元数据**：不只存储 input/output，还分别记录 `cache_read` / `cache_write` / `reasoning` token，以及计费路由（provider + base_url + mode）、估算费用 + 实际费用 + 费用状态 + 价格来源 + 定价版本，从 API 响应到数据库落地形成完整的计费审计链。

**推理链持久化（v6）**：多轮 reasoning 场景下（OpenRouter / OpenAI o 系列 / Nous），assistant 消息的 `reasoning` / `reasoning_details` / `codex_reasoning_items` 字段被持久化，`get_messages_as_conversation()` 还原时将其塞回消息体，让 provider 能重放完整的推理上下文，保持多轮推理连贯性。

**HERMES_HOME 单一入口 + 向后兼容路径解析**：`get_hermes_home()` 是所有路径的起点，`get_hermes_dir()` 同时支持新旧两种子目录布局（旧路径存在就用旧的，不做强制迁移），避免升级时破坏已有目录结构。

**凭证池的按错误码差异化冷却 + Provider 提供的重置时间戳优先**：普通限流（429）只冷却 1 小时，欠费/配额耗尽（402）冷却 24 小时——配额重置节奏不同，统一时长要么太长要么太短。更重要的是，`_exhausted_until()` 优先读取 Provider 在错误响应中明示的 `reset_at` 字段（支持 epoch 秒/毫秒/ISO-8601 + 错误消息中的 `quotaResetDelay` 文本），以实际重置时刻为准，完全消除无谓的提前等待。

**OAuth 令牌跨进程单次消费同步**：Anthropic / Codex OAuth 的 refresh_token 是单次使用的。当 Claude Code CLI 或另一个 Profile 先行刷新后，当前 Pool 条目的 refresh_token 立即失效。`_sync_anthropic_entry_from_credentials_file()` 和 `_sync_codex_entry_from_cli()` 在凭证被标记 exhausted 时主动检查外部凭证文件，发现令牌已更新则直接同步（不请求 Auth Server），将跨进程的令牌竞争转化为无感知的自动恢复。

**Profile = HERMES_HOME 的环境变量隔离**：Hermes 通过在 CLI 入口点 import 任何业务模块之前设置 `HERMES_HOME` 环境变量来实现 Profile 隔离。这意味着隔离完全不依赖依赖注入或全局对象——所有模块的路径解析天然从 `HERMES_HOME` 派生，Profile 切换等同于更换根目录。Wrapper 别名（`coder chat`）让用户无需记忆 `-p` 旗标，`~/.hermes/active_profile` 文件则提供跨终端的粘性默认。

**import 时快照防止脱敏被运行时绕过**：`_REDACT_ENABLED` 在模块 import 时立即计算并缓存，与之后的 `os.environ` 变化解耦。即使 LLM 在对话中生成了 `export HERMES_REDACT_SECRETS=false` 并由 shell 执行，当前进程的脱敏开关也不受影响，防止任意代码执行漏洞静默地打开日志记录敏感信息。

## 局限性

**单 SQLite 文件的写吞吐上限**：WAL 模式只允许一个写者，多进程高并发写入（gateway 处理大量并发消息）时，即使有抖动重试，15 次重试（总计约 2.25s 最大等待）仍然可能耗尽，在极端场景下抛出 `database is locked`。批量导入或压测场景需要额外考量。

**FTS5 索引与主表的一致性窗口**：FTS5 通过触发器同步，触发器在同一事务内执行，理论上是一致的。但 `_init_schema()` 对 FTS 表存在性检测用的是 `SELECT * FROM messages_fts LIMIT 0`——如果 FTS 表损坏或部分存在，这里不能完整重建，需要手动介入。

**token 计数的 CLI/Gateway 二元模式复杂度**：`update_token_counts()` 的 `absolute` 参数区分 CLI（增量累加）和 Gateway（绝对覆盖）两种语义，调用方需要清楚知道自己处于哪种模式，否则容易产生重复计数（CLI 多次调用）或丢失计数（Gateway 误用增量模式）的 bug。

**session_id 格式无校验**：`session_id` 是纯字符串主键，格式约定（`YYYYMMDD_HHMMSS_hex6`）仅靠生成逻辑约束，没有表级约束或正则验证。外部传入非标准格式的 session_id 不会报错，但会造成前缀搜索（`resolve_session_id()`）的语义失效。

**标题全局唯一性的弱保障**：`set_session_title()` 在写事务内做唯一性检查，但 `get_next_title_in_lineage()` 的编号查询是读锁之外的操作，高并发下两个进程可能各自算出相同的 `#N`，导致后写入者因唯一索引冲突失败，标题不续号而直接丢弃。

## 来源

- 源码版本：Hermes Agent 0.16.0
- 分析文件：`hermes_state.py`、`hermes_constants.py`、`hermes_logging.py`、`hermes_time.py`、`run_agent.py`（AIAgent 初始化 / `_flush_messages_to_session_db` / 压缩分裂 / token 更新路径）、`agent/usage_pricing.py`、`agent/title_generator.py`、`hermes_cli/config.py`、`agent/credential_pool.py`、`hermes_cli/profiles.py`、`agent/redact.py`
- 分析深度：源码级
