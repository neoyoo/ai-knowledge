---
title: Channel & Remote
aliases: [渠道, remote execution, multi-channel]
category: L1
created: 2026-04-06
updated: 2026-04-08
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[runtime-state]]"
    type: uses
sources: [claude-code, openharness, deer-flow, hermes-agent]
---

## 一句话定义

多渠道接入（CLI、Web、Slack、IDE）与远程执行能力。

## 核心问题

- 不同渠道的输入输出格式差异怎么抹平？
- 远程执行和本地执行的安全模型有什么区别？
- 渠道间的会话状态怎么同步？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|-------------|
| 核心设计 | Channel（MCP 子协议，解决外部消息通道异步接入）+ Remote Session（把云端 agent 实例折叠回本地 task 系统统一编排），两者共同将 agent 从终端内对话循环扩展为跨设备可恢复 agent runtime | Python 后端 + React/Ink TUI 前端通过 stdio JSON 协议通信的混合架构；`BridgeSessionManager` 管理长生命周期子进程；cron 系统通过 `RemoteTriggerTool` 支持定时触发 agent 执行；当前"远程"本质上是本地子进程管理 | Nginx 反向代理统一入口（`:2026`）三服务架构（LangGraph Server + FastAPI Gateway + Next.js），内置 Slack/Telegram/Feishu/WeCom 四个 IM 平台适配器，同时提供嵌入式 Python SDK（`DeerFlowClient`）和 ACP 协议跨 harness 互调 | `GatewayRunner` 单 asyncio 事件循环并发驱动 16 个平台适配器；`BasePlatformAdapter` 三抽象方法 + 可选 override 接口统一所有平台差异；session key 三维语义（DM/群组/thread，thread 默认共享）；`SessionResetPolicy` 四模式 + memory flush 前置；cron 文件锁 + SILENT_MARKER + Matrix E2EE 优先投递；6 种执行后端（local/Docker/SSH/Modal/Daytona/Singularity）|
| 关键特点 | Channel 复用整套 MCP 基础设施（无需单独发明 IM 插件 runtime）；权限 relay 走结构化 typed notification 而非文本 regex（防止自然语言误触发）；`RemoteAgentTask` 把远程 session 折叠为本地 task，支持 `--resume` 跨 CLI 重启 | 混合语言架构（Python + Node.js React/Ink），stdio JSON 协议解耦，各语言专注擅长领域；cron 调度内置，JSON 存储简单可审计；`WorkSecret` 凭证编码为未来网络化扩展预留接口形态 | 最完整的 channel 覆盖（4 个 IM 平台 + Web UI + Python SDK + ACP）；嵌入式 SDK 无服务器运行（`DeerFlowClient` 零基础设施启动）；ACP 协议实现跨 harness 互操作（可作为 meta-orchestrator 调度外部 agent）；Nginx 统一入口简化运维 | 16 个平台 + 6 个执行后端，覆盖面最广；智能模型路由（每条消息自动在廉价/强模型间切换）；`_agent_cache` 按 session key 缓存 agent 实例（保全 prefix cache，避免 ~10x 额外费用）；平台感知 PII 脱敏（WhatsApp/Signal/Telegram 启用，Discord 排除保留 `<@user_id>` mention 语义）；typing 暂停机制解决 Slack 输入框锁定问题 |
| 局限 | Channel 依赖 Claude.ai OAuth，纯 API key 用户无法使用；Remote session 要求 git repo + git remote；远程 session 只能 HTTP 轮询不能 WebSocket 推送 | 无 WebSocket 服务端，无 OAuth，无云端 channel；所谓"远程"是本地子进程管理；`WorkSecret` 机制已设计但未连接任何活跃网络端点；stdio JSON 通信在进程异常退出时缺乏健壮的重连机制 | Channel 适配器偏薄（仅处理纯文本，不支持富交互元素）；SSE 单向推送（无 WebSocket 双向通信）；无 per-channel 权限模型；IM 平台依赖平台侧 Webhook，本地开发需 ngrok 中转 | 所有适配器共享单进程 asyncio loop，WhatsApp Node.js bridge 阻塞可影响其他平台延迟；session key 不跨平台（同一用户 Telegram/Discord 是两个独立 session）；`_agent_cache` 无 LRU/TTL 驱逐，大量用户场景内存持续增长；cron 文件锁无法水平扩展到多机；所有会话共享同一 `TERMINAL_ENV`，无法为不同用户或平台配置不同执行后端 |

## 设计权衡

### 方案对比

| 方案 | 核心机制 | 接入灵活性 | 实现复杂度 | 典型场景 |
|------|---------|-----------|-----------|---------|
| 单渠道（CLI only） | 一个入口，一套逻辑 | 低 | 极低 | 开发者工具、个人助手 |
| 多渠道适配器（Channel Adapters） | 统一 core engine，多个 interface adapter | 中（受限于最弱渠道的能力） | 中（每个 adapter 独立维护） | 团队内部工具、多场景产品 |
| Headless API + 独立前端 | Agent 暴露纯 REST/WebSocket API，前端独立构建 | 高（前端任意技术栈） | 高（API 设计、认证、限流） | 平台产品、第三方集成 |
| 混合架构（Backend + Detachable TUI） | Python 后端 + Node.js React TUI 通过 stdio JSON 协议通信（OpenHarness 方案） | 中（TUI 可替换，但协议耦合） | 中高（两个 runtime + 协议维护） | 需要精美终端 UI 的开发工具 |
| 单进程多平台网关（Gateway Runner） | 单 asyncio 事件循环并发驱动所有平台适配器，统一处理会话/安全/路由（Hermes Agent 方案） | 高（适配器可独立插拔，新平台只需实现三个方法） | 高（16 个平台适配器 + 重连/cron/session 管理全部集中） | 需要同时在多个 IM 平台上部署同一 agent 的个人/小团队 |

### 场景决策指南

**个人 CLI 工具 / 开发者自用** → 单渠道。多渠道是过度设计。一个入口，最快上线，最少维护。

**团队内部工具（Slack + Web 并存）** → 多渠道适配器。关键是先把 core engine 设计好，adapter 层只负责格式转换和事件映射，不含业务逻辑。注意：功能集要取所有渠道的交集，Slack 特有的 interactive buttons 在 Web 端可能无等价物，反之亦然。

**面向开发者的 API 产品** → Headless API。前端独立演化，第三方可以自由集成。代价是 API 设计必须足够稳定：一旦对外发布，破坏性变更成本极高。先把 API 设计对，再上 SDK。

**需要精美终端 UI 的开发工具** → 混合架构（参考 OpenHarness）。Python 做 agent 逻辑（生态丰富），Node.js/React/Ink 做 TUI（UI 组件丰富）。stdio JSON 协议解耦两端，TUI 甚至可以换成 Web UI 而不动后端。前提：两个进程的生命周期管理和重连逻辑必须写稳。

**需要跨设备、跨 CLI 重启继续工作**（Claude Code Remote Session 场景）→ 考虑把远程 session 折叠为本地 task（`RemoteAgentTask` 模式），使 `--resume` 等本地能力自然适用，而不是为远程单独建一套编排路径。

**需要同时接入多个 IM 平台（Slack + Telegram + Discord + 企业微信等）** → 单进程多平台网关（参考 Hermes Agent）。把所有平台差异下沉到 `BasePlatformAdapter`（三个抽象方法 + 可选 override），runner 层统一处理安全、会话、路由。关键：session key 的 DM/群组/thread 三维语义必须设计清楚——群组内是否 per-user 隔离、thread 是否共享都影响多人协作体验。

**需要定时执行 agent 任务并投递到 IM 平台** → cron + 平台 live adapter 优先投递。纯 HTTP 投递无法处理 Matrix E2EE 房间（没有密钥），必须通过已建连的 live adapter 发送。同时引入文件锁防止 gateway、daemon、systemd timer 三者同时触发 tick（`fcntl` 文件锁，单机串行保证）。需要审计但不想打扰用户的任务用 `SILENT_MARKER` 标记：agent 输出本地保存，但不向平台投递。

### 常见陷阱

**多渠道没有统一权限模型**：CLI 有沙箱和工具确认机制，Slack 适配器如果直接透传命令，等于从 IM 绕过了安全层。解法：权限检查必须在 core engine 层做，不能依赖各 adapter 自己实现。Claude Code 的权限 relay 走结构化 typed notification 而非文本 regex，就是为了防止自然语言消息误触发高权限操作。

**API 没有 rate limiting，被滥用或意外超额**：Headless API 对外暴露后，没有限流等于裸奔。高并发下 LLM API 费用会在分钟级爆表。最低要求：按 API key 限制 RPM 和 TPM，超额返回 429。

**适配器层太薄，渠道特有能力被浪费**：多渠道适配器取交集意味着放弃了各渠道的差异化能力——Slack 的 Block Kit、Web 的富文本、CLI 的流式输出各有独特价值。如果产品中某个渠道是主渠道，可以为该渠道做「增强适配器」，在通用接口之上支持渠道原生能力，而不是强制所有渠道降级到最低公分母。

**stdio 协议在子进程异常退出时缺乏重连**：OpenHarness 的混合架构依赖 stdio JSON 通信，若 TUI 进程意外崩溃，后端 `BridgeSessionManager` 没有主动重连逻辑，会导致会话静默丢失。解法：后端监听子进程 `exit` 事件，自动重启或向用户暴露错误，而不是静默挂起。

**平台感知的 PII 脱敏被遗漏**：多平台场景下，同一份脱敏策略不能一刀切。Discord 的 mention 语法 `<@user_id>` 要求 LLM 持有真实 user_id，若错误脱敏则 @ 用户失效。解法：维护 `_PII_SAFE_PLATFORMS` 白名单（如 WhatsApp/Signal/Telegram），仅对这些平台做 SHA256 哈希脱敏，Discord 等依赖真实 ID 的平台明确排除。脱敏只影响 LLM 输入（system prompt），路由层始终使用原始 ID。

**Agent 实例重建导致 prompt cache 命中率骤降**：每条消息重新创建 `AIAgent` 实例，会使支持 prefix caching 的提供商（如 Anthropic）的系统提示缓存完全失效，导致 token 费用约增加 10 倍。解法：按 session key 缓存 agent 实例（`_agent_cache`），缓存失效条件改为 config 签名变化（config.yaml 改动），而非时间 TTL 或请求次数。

**Session 重置前未 flush 记忆导致上下文丢失**：daily/idle 重置触发时，若直接清空对话历史，用户之前提到的偏好、任务状态等重要信息随之消失。解法：重置前异步启动轻量 `AIAgent`，让它审阅对话历史，把重要内容写入 `MEMORY.md`/`USER.md`，flush 结果持久化到 `sessions.json`（`memory_flushed=True`），防止 gateway 重启后重复 flush。连续失败 3 次后主动放弃，避免无限重试阻塞重置流程。

## L2 详情

- [[channel-remote--claude-code]]
- [[channel-remote--openharness]]
- [[channel-remote--deer-flow]]
- [[channel-remote--hermes-agent]]
