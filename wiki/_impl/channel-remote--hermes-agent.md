---
title: "Channel & Remote — Hermes Agent"
category: L2
parent: "[[channel-remote]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的 Channel & Remote 实现是目前开源 agent 项目中平台覆盖最广、工程化程度最高的多通道网关方案。它以 `GatewayRunner`（`gateway/run.py`）为核心，通过统一的 `BasePlatformAdapter` 接口把 16+ 个 IM/消息平台统一接入，以 `SessionStore` + `SessionResetPolicy` 管理跨平台会话生命周期，以 `cron/scheduler.py` 实现持久化定时任务投递，并通过 `TERMINAL_ENV` 环境变量在 local/Docker/SSH/Modal/Daytona/Singularity 六种执行后端之间切换。最新版又补强了 TUI/Desktop 的 route resume、gateway sleep/wake 后的 session rebind，以及 draft/edit/fallback 多层流式投递。整体设计思路：**平台差异被适配层消化，业务逻辑（会话、路由、安全、恢复）集中在 runner/gateway 层统一处理**。

## 架构分析

### 1. GatewayRunner：单一事件循环驱动多适配器

`GatewayRunner`（`gateway/run.py`）是整个 channel 系统的根对象，在一个 asyncio 事件循环中并发驱动所有平台适配器。

**启动流程**（`GatewayRunner.start()`）：
1. 从 `~/.hermes/config.yaml` 加载 `GatewayConfig`，初始化 `SessionStore`、`DeliveryRouter`、`PairingStore`、`HookRegistry`
2. 遍历 `config.platforms`，对每个 enabled 平台调用 `_create_adapter()` 工厂方法，注册 `_handle_message` 和 `_handle_adapter_fatal_error` 回调
3. 所有适配器 `await adapter.connect()` 并发启动（各平台的 event loop 接入方式不同，但都在同一个 asyncio loop 内协作）
4. 启动三个后台任务：`_session_expiry_watcher`（每 5 分钟扫描过期 session，触发 memory flush）、`_platform_reconnect_watcher`（断线重连，指数退避 30s→60s→120s→300s 上限，最多 20 次）、process watcher（监控后台进程输出）

**适配器重连机制**（`_platform_reconnect_watcher`）：平台断连后进入 `_failed_platforms` 字典，存储 `{config, attempts, next_retry}`，后台定时检查，指数退避重试，非 retryable 错误立即放弃，retryable 错误（如网络波动）持续重试。

**智能路由**（`_resolve_turn_agent_config`）：每次 agent 调用前通过 `agent.smart_model_routing` 模块，根据消息内容自动在廉价模型与强模型之间路由，无需用户手动切换。

**Agent 缓存**（`_agent_cache`）：每个 session_key 缓存一个 `AIAgent` 实例，避免每条消息重建 agent（重建会丢失系统提示缓存，在支持 prefix caching 的 Anthropic 提供商下成本高 ~10x）。

### 2. BasePlatformAdapter：统一适配器契约

`BasePlatformAdapter`（`gateway/platforms/base.py`）定义了所有平台适配器必须实现的接口：

**抽象方法**：
- `connect() → bool` — 建立连接
- `disconnect()` — 断开连接
- `send(chat_id, content, reply_to, metadata) → SendResult` — 发送消息

**可选重写方法**（默认 fallback 实现）：
- `edit_message()` — 编辑已发送消息（流式输出用）
- `send_typing()` / `stop_typing()` — 输入状态指示器
- `send_image()` / `send_image_file()` / `send_animation()` — 原生图片投递
- `send_voice()` / `play_tts()` — 语音/TTS 投递
- `send_video()` / `send_document()` — 视频和文件附件
- `on_processing_start()` / `on_processing_complete()` — 生命周期钩子（Discord 用此添加 👀/✅/❌ 反应）

**消息模型**（`MessageEvent` dataclass）：
```python
text: str
message_type: MessageType  # TEXT/LOCATION/PHOTO/VIDEO/AUDIO/VOICE/DOCUMENT/STICKER/COMMAND
source: SessionSource       # 来源标识
media_urls: List[str]       # 本地缓存路径（下载后）
reply_to_message_id: str    # 回复上下文
auto_skill: Optional[str]   # 频道绑定自动加载的 skill
```

**发送结果模型**（`SendResult`）：
```python
success: bool
message_id: Optional[str]
error: Optional[str]
retryable: bool  # True = 可重试的瞬态网络错误
```

**重试机制**（`_send_with_retry`）：遇到 `retryable` 错误（`connecterror`/`connectionreset`/`broken pipe`/`eoferror` 等）触发指数退避重试（`base_delay=2.0` 秒，`max_retries=2`）；超时错误（`readtimeout`/`writetimeout`）不重试（请求可能已送达）；非网络格式错误 fallback 到纯文本版本再发一次。所有重试耗尽后向用户发送"投递失败"通知。

**媒体本地缓存**：平台图片/音频/文档 URL 有时效性（如 Telegram 文件 URL ~1 小时过期），适配器收到媒体消息后立即下载到本地缓存：
- 图片 → `~/.hermes/cache/images/img_{uuid12}.jpg`
- 音频 → `~/.hermes/cache/audio/audio_{uuid12}.ogg`
- 文档 → `~/.hermes/cache/documents/doc_{uuid12}_{原始文件名}`

下载函数（`cache_image_from_url`/`cache_audio_from_url`）含 SSRF 防护（`tools.url_safety.is_safe_url` 检查），阻止访问内网地址。

**连续输入指示器**（`_keep_typing`）：每 2 秒重发 typing indicator（Telegram/Discord 状态 ~5 秒过期），直到 handler 完成。Slack 的 `assistant_threads_setStatus` 会禁用输入框，因此引入 `_typing_paused` 机制，在等待危险命令审批时暂停 typing，让用户能输入 `/approve` 或 `/deny`。

**媒体提取**（`extract_media`/`extract_images`/`extract_local_files`）：从 agent 响应中提取：
- `MEDIA:<path>` 标签（TTS 工具专用）
- Markdown 图片 `![alt](url)` 和 HTML `<img src="url">`
- 裸本地文件路径（绝对路径 + 媒体扩展名）

提取后通过平台原生 API 发送为附件，文本中的路径/标签同时清除。

### 2.5 TUI/Desktop route resume

Hermes 的桌面/TUI 不把 URL route 直接绑定到当前 live runtime id，而是使用持久化 session id，再通过 gateway JSON-RPC 重新绑定：

- `tui_gateway/server.py:3422-3463` — `session.list` deny-list 内部 tool session，面向用户展示所有 human-facing session；
- `tui_gateway/server.py:3512-3649` — `session.resume` 支持按 id/title 恢复，profile scoped state.db，已有 live session 走 fast path；构造 agent 时释放 resume lock，完成后 double-check 防并发 resume；
- `apps/desktop/src/app/routes.ts:68-80` — `/:sessionId` 解析为 route session id；
- `apps/desktop/src/app/session/hooks/use-route-resume.ts:72-104` — route change、gateway became open、stranded routed session 三类情况触发 resume；
- `use-session-actions.ts:540-590` — 先读本地 snapshot，再调用 gateway `session.resume`，避免刷新时消息列表清空闪烁；
- `use-prompt-actions.ts:490-510` — `prompt.submit` 遇到 `session not found` 时自动 `session.resume` 并用新的 live id 重试一次。

这条链路解决的是 Web/Desktop agent 常见 split-brain：前端还停在历史 session route，但 gateway 后端重启、profile swap 或 sleep/wake 后 live runtime id 已失效。

### 2.6 Gateway restart resume_pending

Gateway 对长任务 shutdown/restart 做 durable resume marker：

- `gateway/session.py:482-492` — `resume_pending` / `resume_reason` / `last_resume_marked_at` 持久化到 session entry；
- `gateway/session.py:1004-1050` — `mark_resume_pending()` 与 `clear_resume_pending()`；
- `gateway/session.py:1110-1144` — startup crash/fast restart 后标记最近活跃 session 为 resumable；
- `gateway/run.py:4375-4460` — adapter online 后调度空文本 internal event 自动续跑 resume-pending session；
- `gateway/run.py:5675-5758` — shutdown/restart drain 前预先写 resume_pending；drain 成功则清除，超时则保留并 interrupt。

这比“进程重启后用户手动 /resume”更接近产品体验：平台 reconnect 后可以只恢复该平台自己的未完成 session，且不会把显式 `/stop` 的 suspended session 误恢复。

### 2.7 StreamConsumer draft/edit/fallback

`gateway/stream_consumer.py` 将同步 agent callback 桥接到异步平台投递，并根据平台能力选择流式策略：

- `StreamConsumerConfig.transport` 支持 `auto` / `draft` / `edit` / `off`；
- native draft streaming 用 `send_draft` 显示中间帧，final answer 仍走真实 sendMessage；
- edit path 支持 flood-control adaptive backoff，连续失败后进入 fallback；
- long-running preview 可用 fresh-final 发送完成态新消息，让平台可见时间戳反映完成时间；
- fallback mode 只发送 missing tail，避免重复整段回复。

这说明 channel 的“流式输出”不是一个开关，而是平台能力、编辑限制、长任务耗时和用户可见性之间的路由策略。

### 3. 平台覆盖：16 个适配器

`gateway/config.py` 的 `Platform` 枚举定义了全部平台：

| 平台 | 适配器文件 | 认证方式 |
|------|-----------|---------|
| Telegram | `telegram.py` + `telegram_network.py` | Bot Token |
| Discord | `discord.py` | Bot Token |
| Slack | `slack.py` | Bolt SDK |
| WhatsApp | `whatsapp.py` | Node.js bridge |
| Signal | `signal.py` | signal-cli HTTP |
| Matrix | `matrix.py` | matrix-nio（支持 E2EE）|
| Mattermost | `mattermost.py` | API Token |
| Feishu（飞书）| `feishu.py` | App ID/Secret |
| WeCom（企业微信）| `wecom.py` | Bot ID/Secret |
| DingTalk（钉钉）| `dingtalk.py` | Client ID/Secret |
| Email | `email.py` | IMAP + SMTP |
| SMS | `sms.py` | Twilio SID |
| Home Assistant | `homeassistant.py` | HASS Token |
| API Server | `api_server.py` | 内置 HTTP 服务 |
| Webhook | `webhook.py` | HMAC 签名验证 |
| 本地 CLI | （非 socket，直接调用）| 无 |

`_create_adapter()` 工厂方法做两件事：检查依赖是否已安装（如 `TELEGRAM_AVAILABLE = try import telegram`），然后 lazy import + 实例化对应适配器。

Telegram 额外有 `telegram_network.py`（`TelegramFallbackTransport`/`discover_fallback_ips`），处理 Telegram 在部分地区被封锁时的 fallback IP 路由。

Matrix 适配器使用 `matrix-nio[e2e]`，支持 E2EE 房间——这是 cron 投递中有特殊意义的平台：E2EE 房间的消息不能通过独立 HTTP 发送（没有密钥），必须通过 live adapter 路径，`cron/scheduler.py` 的 `_deliver_result` 函数会优先检查 live adapter 是否存在。

### 4. SessionSource 与会话路由

`SessionSource`（`gateway/session.py`）是会话路由的核心数据结构：

```python
@dataclass
class SessionSource:
    platform: Platform
    chat_id: str
    chat_name: Optional[str]
    chat_type: str        # "dm" / "group" / "channel" / "thread"
    user_id: Optional[str]
    user_name: Optional[str]
    thread_id: Optional[str]  # Discord threads, Telegram 话题, Slack threads
    chat_topic: Optional[str] # Discord/Slack 频道 topic
    user_id_alt: Optional[str] # Signal UUID（备选 ID）
    chat_id_alt: Optional[str] # Signal group 内部 ID
```

**Session key 构建**（`build_session_key`）——决定消息属于哪个 session 的核心逻辑：

- **DM 模式**：`agent:main:{platform}:dm:{chat_id}` 或含 `:{thread_id}`
- **群组/频道**：`agent:main:{platform}:{chat_type}:{chat_id}[:{thread_id}][:{user_id}]`

关键隔离规则：
- `group_sessions_per_user=True`（默认）：群组内每个用户独立 session，防止不同用户的对话历史混淆
- `thread_sessions_per_user=False`（默认）：thread 内所有用户共享同一 session（符合 Telegram forum topic、Discord thread、Slack thread 的使用语义，团队对话不需要每人一份上下文）
- Signal 支持双 ID（phone number + UUID），`user_id_alt` 提供备选匹配

**动态 system prompt 注入**（`build_session_context_prompt`）：每个 session 的 system prompt 包含当前上下文（来源平台、chat_id、用户名、连接的平台列表、home channel 信息、可用的投递目标选项）。平台特定警告也在此注入，例如 Slack 和 Discord 适配器会告知 agent 它**不能**调用平台特定 API（搜索历史、置顶消息、管理频道等），防止 agent 做出无法兑现的承诺。

### 5. Session 重置策略

`SessionResetPolicy`（`gateway/config.py`）定义四种模式：

| 模式 | 说明 |
|------|------|
| `daily` | 每天 `at_hour` 时刻（默认凌晨 4 点）重置 |
| `idle` | 超过 `idle_minutes`（默认 1440 分钟 = 24 小时）无活动重置 |
| `both` | `daily` 和 `idle` 任意一个触发即重置（默认）|
| `none` | 永不自动重置（上下文压缩管理） |

重置时 `notify=True` 时向用户发送通知（`api_server`/`webhook` 平台默认排除）。支持按 chat_type 或 platform 分别配置不同策略。

**Memory flush 前置**（`_flush_memories_for_session` / `_session_expiry_watcher`）：session 过期前（每 5 分钟 watcher 检查），异步在线程池中运行一个轻量 `AIAgent`，让它审阅对话历史，把重要内容写入 `MEMORY.md`/`USER.md`，并考虑是否创建新 skill。flush 完成后在 `SessionEntry.memory_flushed=True` 并持久化到 `sessions.json`，防止 gateway 重启后重复 flush。连续失败 3 次后标记 `memory_flushed=True` 放弃（避免无限重试）。

### 6. PII 脱敏机制

`gateway/session.py` 实现轻量但系统化的 PII 脱敏：

```python
def _hash_id(value: str) -> str:
    """SHA256 截取 12 hex 位，确定性哈希"""
    return hashlib.sha256(value.encode()).hexdigest()[:12]

def _hash_sender_id(value: str) -> str:
    return f"user_{_hash_id(value)}"

def _hash_chat_id(value: str) -> str:
    """保留平台前缀，只哈希数字部分"""
    # "telegram:12345" → "telegram:<hash>"
```

脱敏仅在 `_PII_SAFE_PLATFORMS`（WhatsApp/Signal/Telegram）上启用，Discord 被排除——因为 Discord 的 mention 语法 `<@user_id>` 需要 LLM 持有真实 user_id 才能正确 @ 用户。脱敏仅影响注入 system prompt 的文本，路由层始终使用原始 ID。

手机号检测：`_PHONE_RE = re.compile(r"^\+?\d[\d\-\s]{6,}$")` 识别 E.164 及类似格式。

### 7. 消息处理管道

`BasePlatformAdapter.handle_message()` + `GatewayRunner._handle_message()` 构成消息处理管道：

**adapter 层**（`handle_message`）：
1. 构建 `session_key`
2. 若该 session 已有活跃 handler：
   - `/approve`/`/deny`/`/status`/`/stop`/`/new`/`/reset` 命令 → bypass guard 直接调用 handler（防止 `/approve` 死锁）
   - 连拍照片 → 合并到 `_pending_messages`，不中断当前 run
   - 普通消息 → 设置 `_active_sessions[session_key]` 的中断事件，挂入 `_pending_messages`
3. 同步设置 `_active_sessions[session_key] = asyncio.Event()`（关闭 TOCTOU 竞态窗口）
4. 创建后台 task `_process_message_background`

**runner 层**（`_handle_message`）：
1. `_is_user_authorized` — 检查 allowlist / pairing store / GATEWAY_ALLOW_ALL_USERS
2. 未授权 DM → 生成 pairing code 回复（`PairingStore.generate_code`）
3. 处理 `/reset`、`/new` 等重置命令
4. 加载或创建 session，注入 session context 到 system prompt
5. 通过缓存的 `AIAgent` 实例执行 `run_conversation()`
6. agent 调用时若发现 session 已过期（daily/idle 触发）→ 先 flush memories，再重置，再注入"已自动重置"通知

**过期 agent 驱逐**：维护 `_running_agents_ts` 记录 agent 启动时间戳，结合 `agent.get_activity_summary()` 获取最近活动时间，空闲超过 `HERMES_AGENT_TIMEOUT`（默认 1800 秒）时驱逐（evict），防止僵尸 agent 持续占用 session。

### 8. Cron 定时任务系统

**持久化存储**（`cron/jobs.py`）：
- 任务持久化到 `~/.hermes/cron/jobs.json`（原子写入 + fsync，权限 0600）
- 输出保存到 `~/.hermes/cron/output/{job_id}/{timestamp}.md`
- 目录权限 0700

**调度格式**（`parse_schedule()`）支持：
- 一次性：`"30m"`/`"2h"`/`"1d"` → 从现在起延迟；`"2026-02-03T14:00"` → 指定时间
- 间隔：`"every 30m"`/`"every 2h"` → 循环
- Cron 表达式：`"0 9 * * *"` → 需要 `croniter` 库

**Tick 循环**（`cron/scheduler.py`）：
- 文件锁（`~/.hermes/cron/.tick.lock`，Unix `fcntl` / Windows `msvcrt`）防止并发 tick（gateway + daemon + systemd timer 可能重叠）
- 每 60 秒 tick 一次，调用 `get_due_jobs()` + `run_job()`
- 一次性任务有 `ONESHOT_GRACE_SECONDS=120` 秒宽限窗口，防止在正点创建的任务因到达时间比 tick 晚几秒而被跳过
- `SILENT_MARKER = "[SILENT]"` 机制：agent 响应以 `[SILENT]` 开头时抑制投递，输出仍本地保存审计

**Job 构成字段**：
- `prompt` — agent 执行的提示词
- `skills` — 执行前加载的 skill 列表
- `script` — Python 脚本路径（仅限 `HERMES_HOME/scripts/` 内），执行后 stdout 注入 prompt 上下文
- `deliver` — 投递目标：`"local"`/`"origin"`/`"telegram"`/`"discord:channel_id"` 等
- `origin` — 任务来源（用于 `deliver=origin` 时回复原来的 chat）
- `model`/`provider` — per-job 模型覆盖

**投递路由**（`_deliver_result`）：
1. 解析 `deliver` 字段 → 具体 `{platform, chat_id, thread_id}`
2. 优先通过 live adapter（`asyncio.run_coroutine_threadsafe`）投递 → 支持 Matrix E2EE
3. live adapter 不可用或失败 → standalone 路径：`asyncio.run(_send_to_platform(...))` 在新 event loop 中发送（若已有 running loop 则用 `ThreadPoolExecutor` 绕过）

脚本执行安全：`_run_job_script` 检查路径必须在 `HERMES_HOME/scripts/` 内（防路径遍历 + 绝对路径注入），脚本 stdout 通过 `agent.redact.redact_sensitive_text` 脱敏后再注入 prompt。

### 9. DM Pairing：用户授权流程

`gateway/pairing.py` 实现了一套基于一次性配对码的用户授权系统，替代静态 allowlist。对于陌生用户发来的 DM，GatewayRunner 的 `_is_user_authorized` 检查若失败，会调用 `PairingStore.generate_code()` 生成配对码并回复用户，由 bot 所有者在 CLI 端 `/approve` 通过。

**安全设计**（遵循 OWASP + NIST SP 800-63-4 指导）：

- **密码学随机码**：码从 32 字符无歧义字母表（排除 `0/O/1/I`）中用 `secrets.choice()` 生成，长度 8 位，暴力破解概率 1/32^8 ≈ 1 in 10¹²
- **有效期**：码固定 1 小时过期（`CODE_TTL_SECONDS = 3600`），过期清理在下次 `generate_code` / `list_pending` 时惰性触发
- **速率限制**：同一用户在同一平台每 10 分钟只能请求一次（`RATE_LIMIT_SECONDS = 600`），防止批量枚举
- **等待队列上限**：每个平台最多同时存在 3 个待审批请求（`MAX_PENDING_PER_PLATFORM = 3`），防止队列污染
- **失败锁定（类 TOTP 机制）**：bot 所有者连续 5 次输入错误码后，该平台锁定 1 小时（`MAX_FAILED_ATTEMPTS = 5`，`LOCKOUT_SECONDS = 3600`），阻止在线猜码攻击
- **文件权限**：所有持久化文件（pending / approved / rate_limits）通过 `_secure_write` 写入——原子替换（`tempfile + os.replace`）+ `fsync` 保证一致性，`chmod 0600` 限制只有 owner 可读写
- **码不进日志**：注释中明确 "Codes are never logged to stdout"，防止日志泄露

**存储结构**（`~/.hermes/pairing/`）：
- `{platform}-pending.json` — 待审批请求（code → {user_id, user_name, created_at}）
- `{platform}-approved.json` — 已授权用户（user_id → {user_name, approved_at}）
- `_rate_limits.json` — 速率限制 + 失败计数 + 锁定时间（统一键前缀 `platform:user_id` / `_failures:platform` / `_lockout:platform`）

**线程安全**：`PairingStore` 通过 `threading.RLock` 保护所有读改写循环（gateway 并发驱动多平台适配器，都共享同一个 `PairingStore` 实例）。

**生命周期**：授权通过的用户会被加入 `{platform}-approved.json`，后续 `is_approved(platform, user_id)` 直接查文件即可；`revoke()` 从 approved 列表中删除；`clear_pending()` 用于 CLI 清空所有待审批队列。

### 10. Voice Mode：语音输入输出子系统

Hermes Agent 的 Voice Mode（`tools/voice_mode.py` + `tools/transcription_tools.py` + `tools/tts_tool.py`）为 CLI 通道提供推按说话（Push-to-Talk）+ TTS 播放的完整语音交互能力，是 CLI 通道的可选音频 I/O 模态。

**架构层次**：

```
CLI (cli.py)
  ├── 录音触发（PTT 按键）
  │     └── AudioRecorder.start(on_silence_stop=callback)
  │           └── sounddevice.InputStream（16 kHz, mono, int16）
  ├── 静音检测 → 自动停录
  │     └── AudioRecorder._callback → RMS 计算 → 静音 3s 触发
  ├── STT（tools/transcription_tools.transcribe_audio）
  │     └── 转录结果注入对话作为用户消息
  └── TTS 播放（tools/tts_tool.text_to_speech_tool）
        └── play_audio_file → sounddevice / afplay / ffplay / aplay
```

**录音管道**（`AudioRecorder`）：

- 使用 `sounddevice.InputStream`（16 kHz / mono / int16，与 Whisper 原生采样率对齐）
- 流实例**一次性创建、永久保活**（`_ensure_stream`），避免 macOS CoreAudio 上反复 close/reopen 导致的死锁
- **智能静音检测**：需先确认 0.3s 持续语音（`_min_speech_duration`）才标记"已说话"，此后静默 3s 才自动停录；识别并容忍语音中的短暂停顿（`_max_dip_tolerance = 0.3s`），防止自然停顿误触发停录
- 最长录音安全上限 120 秒（`MAX_RECORDING_SECONDS`），无语音等待 15s 后自动放弃
- 停录时丢弃过短（< 0.3s）或过安静（峰值 RMS < 200）的录音
- **Whisper 幻觉过滤**（`is_whisper_hallucination`）：对 Whisper 在近静音音频上的常见幻觉词组（"Thank you."、"Thanks for watching." 等，含多语言变体）进行精确匹配 + 正则重复模式过滤，过滤后返回空转录而非错误

**STT 提供商**（`transcription_tools.transcribe_audio`，provider 优先级顺序）：

| 提供商 | 实现 | 所需条件 |
|--------|------|---------|
| `local` | `faster-whisper`（CPU/GPU 本地推理）| `pip install faster-whisper` |
| `local_command` | 本地 `whisper` CLI 二进制（ffmpeg 转码非 WAV 格式）| 安装 whisper CLI |
| `groq` | Groq Whisper API（`whisper-large-v3-turbo`，免费额度）| `GROQ_API_KEY` |
| `openai` | OpenAI Whisper API（`whisper-1` / `gpt-4o-transcribe`，付费）| `OPENAI_API_KEY` 或 managed gateway |

无显式配置时自动探测（local > groq > openai）；显式配置 `stt.provider` 则强制使用，不做静默回退。本地 `faster-whisper` 模型单例（`_local_model`）全进程共享，避免重复加载（~150 MB for base 模型）。支持格式：mp3/mp4/mpeg/mpga/m4a/wav/webm/ogg/aac，上限 25 MB。

**TTS 提供商**（`tts_tool.text_to_speech_tool`）：

| 提供商 | 特点 | 所需条件 |
|--------|------|---------|
| `edge`（默认）| 微软 Edge 神经语音，免费 | `pip install edge-tts` |
| `elevenlabs` | 高质量/声音克隆，付费 | `ELEVENLABS_API_KEY` |
| `openai` | `gpt-4o-mini-tts`，付费 | `OPENAI_API_KEY` |
| `minimax` | 高质量含声音克隆（`speech-2.8-hd`），付费 | `MINIMAX_API_KEY` |
| `neutts` | 设备本地 TTS（`neutts_cli`），免费 | 安装 `neutts` |

TTS 输出格式根据目标平台自动切换：Telegram 语音气泡需要 Opus（`.ogg`），通过 `ffmpeg` 从 MP3 转换；CLI 和 Discord/WhatsApp 使用 MP3。音频播放优先尝试 `sounddevice`（WAV），依次 fallback 到 `afplay`（macOS）/ `ffplay`（跨平台）/ `aplay`（Linux ALSA），全部支持 `stop_playback()` 中断。

**环境感知**（`detect_audio_environment`）：自动检测 SSH / Docker / WSL 环境并给出阻断警告；WSL 有 `PULSE_SERVER` 时降级为提示而非阻断（PulseAudio bridge 可用）。`check_voice_requirements()` 汇总 audio capture、STT provider、环境检测三项状态，供 CLI `/voice on` 命令显示诊断信息。

**与 gateway 的关系**：Voice Mode 仅服务于 CLI 通道（`cli.py`）的输入端（麦克风 → 文字）和输出端（文字 → 扬声器）。`BasePlatformAdapter` 已有独立的 `send_voice()` / `play_tts()` 抽象方法，用于 IM 平台（如 Telegram）将 agent 回复投递为语音气泡，与 CLI Voice Mode 共享 TTS 工具层但调用路径不同。

### 11. CLI TUI

`cli.py` 基于 `prompt_toolkit` 构建固定底部输入框的 TUI：
- `Application` + `Layout`（`HSplit` + `Window`）+ `FormattedTextControl` — 纯 terminal 布局
- `FileHistory` — 历史命令持久化（`~/.hermes/history`）
- `KeyBindings` — 可自定义快捷键，`CursorShape.BLOCK` 非闪烁光标
- `_COMMAND_SPINNER_FRAMES` — 等待时的 spinner 动画
- `patch_stdout` — 防止 agent 输出破坏 TUI 布局

Banner 展示：连接后端、token 用量（`CanonicalUsage`/`format_token_count_compact`/`format_duration_compact`）、当前终端后端类型（`TERMINAL_ENV`）及配置（SSH host、Docker image 等）。

### 10. 执行后端：6 种 Terminal 环境

`TERMINAL_ENV` 环境变量控制 agent 使用哪个终端后端（`tools/terminal_tool.py`）：

| 后端 | 关键配置 | 说明 |
|------|---------|------|
| `local` | 无额外配置 | 直接在主机执行，默认 cwd = 当前目录 |
| `docker` | `TERMINAL_DOCKER_IMAGE` | Docker 容器，默认 cwd = `/root` |
| `ssh` | `TERMINAL_SSH_HOST`/`USER`/`PORT`/`KEY` | 远程 SSH，`SSHEnvironment`（`tools/environments/ssh.py`）|
| `modal` | `TERMINAL_MODAL_IMAGE` | Modal serverless sandbox |
| `daytona` | `TERMINAL_DAYTONA_IMAGE`/`DAYTONA_API_KEY` | Daytona cloud dev environment，`DaytonaEnvironment`（`tools/environments/daytona.py`）|
| `singularity` | `TERMINAL_SINGULARITY_IMAGE` | HPC 集群 Singularity container，`SingularityEnvironment`（`tools/environments/singularity.py`）|

主机路径处理：container 后端（docker/modal/singularity/daytona）若 `TERMINAL_CWD` 是主机路径（`/Users/`/`/home/` 前缀），自动忽略并回退到 `/root`，防止容器内路径不存在报错。Docker 支持显式 `TERMINAL_DOCKER_MOUNT_CWD_TO_WORKSPACE=true` 将主机 cwd 挂载到 `/workspace`。

在 RL 训练环境（`environments/hermes_base_env.py`）中，后端选择通过 `HermesAgentEnvConfig` 配置，支持 Atropos 集成，可在 Atropos managed server 内以分布式方式运行 agent rollout。

## 关键代码路径

- `gateway/run.py` — `GatewayRunner`：启动/停止/消息处理/adapter 管理/重连/session 过期
- `gateway/session.py` — `SessionSource`/`SessionEntry`/`SessionStore`/`build_session_key`/PII 脱敏/`build_session_context_prompt`
- `gateway/config.py` — `Platform`/`SessionResetPolicy`/`GatewayConfig`/`StreamingConfig`/`HomeChannel`
- `gateway/platforms/base.py` — `BasePlatformAdapter`/`MessageEvent`/`SendResult`/重试/媒体缓存/SSRF 防护
- `gateway/platforms/{telegram,discord,slack,whatsapp,...}.py` — 各平台适配器实现
- `gateway/platforms/telegram_network.py` — Telegram fallback IP transport
- `cron/jobs.py` — job CRUD/`parse_schedule`/`compute_next_run`/安全文件写入
- `cron/scheduler.py` — `tick()`/`run_job()`/`_deliver_result`/file lock/SILENT_MARKER
- `gateway/pairing.py` — `PairingStore`：配对码生成/审批/速率限制/锁定/文件原子写入
- `cli.py` — prompt_toolkit TUI 实现
- `tools/voice_mode.py` — `AudioRecorder`：录音/静音检测/幻觉过滤；`play_audio_file`：跨平台音频播放；`detect_audio_environment`：环境检测
- `tools/transcription_tools.py` — `transcribe_audio()`：多 provider STT 分派；`_get_provider()`：provider 自动探测逻辑
- `tools/tts_tool.py` — `text_to_speech_tool()`：多 provider TTS 生成；`_convert_to_opus()`：Telegram 语音气泡格式转换
- `tools/terminal_tool.py` — `_get_env_config()`，`TERMINAL_ENV` 分派逻辑
- `tools/environments/ssh.py` — `SSHEnvironment`
- `tools/environments/daytona.py` — `DaytonaEnvironment`
- `tools/environments/singularity.py` — `SingularityEnvironment`

## 设计亮点

**1. 适配器抽象彻底，平台差异完全下沉**：`BasePlatformAdapter` 通过 `send()`/`connect()`/`disconnect()` 三个抽象方法 + 一批可选 override 方法，做到业务层代码对 16 个平台无感。新平台只需实现三个方法，其余（重试、媒体提取、typing 管理、生命周期钩子）全部继承。

**2. session_key 构建的语义精确性**：DM vs 群组、是否含 thread、group session per-user vs 共享 thread，三个维度独立可配置，且有清晰的语义解释（thread 默认共享是因为团队对话语义要求大家看同一份历史）。Signal 的双 ID 支持也在此统一处理。

**3. Agent 缓存的 prompt cache 保全**：`_agent_cache` 是对支持 prefix caching 的提供商（Anthropic）的关键优化，避免每条消息重建 agent 导致 prompt cache miss。缓存失效条件是 config 签名变化（config.yaml 改动），而非时间 TTL。

**4. Cron 文件锁 + E2EE 感知投递**：tick 的 `fcntl` 文件锁解决了 gateway、daemon、systemd timer 三者可能同时触发 tick 的问题。`_deliver_result` 优先走 live adapter 则解决了 Matrix E2EE 房间（独立 HTTP 无密钥无法发送）的投递问题。

**5. typing 暂停机制**（`_typing_paused`）：Slack 的 `assistant_threads_setStatus` 会锁定输入框，因此引入 typing 暂停，让用户能在等待命令审批时输入 `/approve` 或 `/deny`，而不会因 typing indicator 持续占据输入框。这是非常细粒度的平台差异处理。

**6. PII 脱敏的平台感知设计**：只在 `_PII_SAFE_PLATFORMS` 脱敏，Discord 明确排除（因 `<@user_id>` mention 需真实 ID），脱敏只影响 LLM 输入，路由始终用原始值。SHA256 截取 12 位既保证单向不可逆，又足够短可读。

**7. Memory flush 前置**：session 重置前异步 flush memories，而不是在用户发消息时同步 flush（避免响应延迟），且 `memory_flushed` 持久化到 `sessions.json` 防止 gateway 重启后重复 flush。

**8. Pairing 的 TOTP 式锁定设计**：DM 配对码系统在失败授权上叠加了类似 TOTP 的平台级锁定（而非用户级）——5 次错误尝试触发整个平台 1 小时锁定。这避免了攻击者通过大量新账号循环枚举的旁路，代价是误锁时 bot 所有者需手动清理。码本身的 32 字符无歧义字母表 + 8 位长度的设计直接对照 NIST SP 800-63-4 的熵要求。

**9. AudioRecorder 流永久保活**：录音流（`sounddevice.InputStream`）在首次 `/voice on` 时创建并全程保持开启，录音停止时只关闭帧收集而不关闭流。这绕开了 macOS CoreAudio 的已知 bug（反复 close/reopen InputStream 会永久挂起），代价是即使不录音时 PortAudio 仍占用麦克风设备。

**10. STT provider 自动探测不做静默 fallback**：`_get_provider` 明确区分"用户显式配置"和"自动探测"两条路径。显式配置某个 provider 但该 provider 不可用时，直接返回 `"none"` 并打印 warning，而不是悄悄换用另一个 provider。这避免了配置失误时用户以为在用本地模型实际在调用付费 API 的陷阱。

## 局限性

**1. 多平台 gateway 单进程设计**：所有平台适配器运行在同一个 asyncio event loop 中，一个平台的阻塞 I/O（如 WhatsApp Node.js bridge 的同步调用）可能影响其他平台的响应延迟。

**2. Session key 不跨平台**：同一用户在 Telegram 和 Discord 上是两个独立 session，没有跨平台身份统一机制，无法在不同平台之间共享对话历史。

**3. WhatsApp 依赖 Node.js bridge**：WhatsApp 适配器通过 Node.js bridge 连接（非官方 API），WhatsApp 反 bot 检测可能导致封号风险，且 bridge 需要独立维护。

**4. Cron 无法水平扩展**：`tick.lock` 文件锁保证单机串行执行，但无法在多机环境下分布式调度 cron 任务。

**5. 流式输出依赖 `edit_message`**：`StreamingConfig.transport="edit"` 通过周期性调用 `edit_message` 实现实时流式输出效果，但平台 API 对编辑频率有限制（如 Telegram `editMessageText` 有 rate limit），且不是所有平台都支持。

**6. Agent 缓存内存增长**：`_agent_cache` 为每个 session key 缓存一个 `AIAgent` 实例，长期运行下（如有大量独立用户的 group bot）内存会持续增长，目前无 LRU 或 TTL 驱逐策略。

**7. 执行后端仅支持单一 `TERMINAL_ENV`**：所有会话共享同一个 `TERMINAL_ENV` 配置，无法为不同平台或用户配置不同后端（例如可信用户用 local，不可信用户用 Docker 沙箱）。

## 来源

- 源码版本：hermes-agent 0.16.0（`pyproject.toml` / `hermes_cli/__init__.py`）
- 分析深度：源码级（`gateway/run.py`/`session.py`/`config.py`/`platforms/base.py`/`gateway/pairing.py`/`cron/scheduler.py`/`cron/jobs.py`/`cli.py`/`tools/terminal_tool.py`/`tools/voice_mode.py`/`tools/transcription_tools.py`/`tools/tts_tool.py`）
