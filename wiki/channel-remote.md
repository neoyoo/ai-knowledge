---
title: Channel & Remote
aliases: [渠道, remote execution, multi-channel]
category: L1
created: 2026-04-06
updated: 2026-06-10
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[runtime-state]]"
    type: uses
  - target: "[[sandbox-isolation]]"
    type: feeds
    evidence: "HTTP/WS channel 暴露后威胁模型从'本机单用户'跳到'多租户 SaaS'，agent 所有代码执行工具必须从 subprocess 弱沙箱升级到容器/微 VM 级隔离；channel 的开放程度直接决定 sandbox-isolation 的档次要求"
sources: [claude-code, openharness, deer-flow, hermes-agent, agentscope]
---

## 一句话定义

多渠道接入（CLI、Web、Slack、IDE）与远程执行能力。

## 核心问题

- 不同渠道的输入输出格式差异怎么抹平？
- 远程执行和本地执行的安全模型有什么区别？
- 渠道间的会话状态怎么同步？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent | AgentScope |
|------|------------|-------------|----------|-------------|-----------|
| 核心设计 | Channel（MCP 子协议，解决外部消息通道异步接入）+ Remote Session（把云端 agent 实例折叠回本地 task 系统统一编排），两者共同将 agent 从终端内对话循环扩展为跨设备可恢复 agent runtime | `src/openharness` core 提供 engine、commands、memory、channels、tasks；`ohmo/` 作为个人 agent 产品壳注入 workspace、personal memory、skills、plugins、gateway 和 session backend；channel core 通过 `MessageBus`/`ChannelManager` 解耦适配器与 agent runtime | Nginx + Frontend + Gateway；Gateway 内嵌 LangGraph-compatible API/runtime，`/api/langgraph/*` 由 nginx 重写到 Gateway；channel 层升级为 `ChannelService` / `ChannelManager` / `MessageBus`，统一处理七类 channel registry、topic/thread、session override、skill whitelist、入站上传和出站 artifact attachment | Gateway/CLI/TUI/Desktop 多入口共用本地 agent core：`GatewayRunner` 负责 16+ 平台适配器和 session reset；`tui_gateway` JSON-RPC 提供 `session.list`/`session.resume`；desktop 路由 `/:sessionId` 自动 resume 并在 gateway sleep/wake 后自愈 | Python 2.x 主线是 FastAPI app factory + session REST/SSE + MessageBus event replay/live fan-out + Redis session lock；旧版 `realtime/`/`tts/` 多模态通道已不代表当前源码 |
| 关键特点 | Channel 复用整套 MCP 基础设施；权限 relay 走结构化 typed notification 而非文本 regex；`RemoteAgentTask` 把远程 session 折叠为本地 task，支持 `--resume` 跨 CLI 重启 | SDK/product 分层清楚：产品人格、个人 workspace、gateway 策略不污染 core runtime；`MessageBus` 用 inbound/outbound queue 把 channel 与 agent 解耦；`ohmo` 每个 chat/thread 维护 runtime bundle 并恢复 session snapshot | ChannelManager 能处理 outputs-only attachment 安全边界、upload、session merge、channel-user override、skill whitelist；`DeerFlowClient` 允许无服务器嵌入式运行；ACP 协议实现跨 harness 互操作 | 多平台覆盖最广；`_agent_cache` 保全 prefix cache；resume_pending 在 gateway 重启/关闭超时后持久化，下次启动或平台重连可自动续跑；desktop 先读本地 snapshot 再 gateway resume，避免路由刷新时空白闪烁；stream consumer 支持 draft/edit/fresh-final/fallback | SSE endpoint 先 replay buffered events 再 live subscribe，并含 heartbeat；Redis MessageBus 用 `SET NX EX` + heartbeat + token release 做 session 分布式锁；workspace/permission context 与 session runtime 绑定 |
| 局限 | Channel 依赖 Claude.ai OAuth，纯 API key 用户无法使用；Remote session 要求 git repo + git remote；远程 session 只能 HTTP 轮询不能 WebSocket 推送 | core `MessageBus` 是进程内 queue，不是分布式 broker；ohmo channel/product 能力不能直接等同于 OpenHarness SDK 能力；远程多用户授权主要在产品壳处理 | 仍非通用 RBAC/ABAC 权限系统；SSE 单向推送；IM 平台依赖平台侧 Webhook，本地开发需 ngrok 中转；artifact/upload 边界需要和 sandbox/workspace 权限联动 | 所有平台适配器仍集中在单进程 gateway；cron/平台 reconnect/resume_pending 的复杂度高；desktop route resume 依赖前端本地状态、gateway JSON-RPC 和 state.db 三方一致 | 当前 Python 2.x 不再提供旧版 realtime/tts 源码路径；Local workspace manager 未用 `user_id` 参与 workdir 隔离；跨系统 AgentCard/A2A/Nacos 能力应看独立 `agentscope-java` |

## 设计权衡

### 方案对比

| 方案 | 核心机制 | 接入灵活性 | 实现复杂度 | 典型场景 |
|------|---------|-----------|-----------|---------|
| 单渠道（CLI only） | 一个入口，一套逻辑 | 低 | 极低 | 开发者工具、个人助手 |
| 多渠道适配器（Channel Adapters） | 统一 core engine，多个 interface adapter | 中（受限于最弱渠道的能力） | 中（每个 adapter 独立维护） | 团队内部工具、多场景产品 |
| Headless API + 独立前端 | Agent 暴露纯 REST/WebSocket API，前端独立构建 | 高（前端任意技术栈） | 高（API 设计、认证、限流） | 平台产品、第三方集成 |
| 混合架构（Backend + Detachable TUI） | Python 后端 + Node.js React TUI 通过 stdio JSON 协议通信（OpenHarness 方案） | 中（TUI 可替换，但协议耦合） | 中高（两个 runtime + 协议维护） | 需要精美终端 UI 的开发工具 |
| 单进程多平台网关（Gateway Runner） | 单 asyncio 事件循环并发驱动所有平台适配器，统一处理会话/安全/路由（Hermes Agent 方案） | 高（适配器可独立插拔，新平台只需实现三个方法） | 高（16 个平台适配器 + 重连/cron/session 管理全部集中） | 需要同时在多个 IM 平台上部署同一 agent 的个人/小团队 |
| SDK core + product shell | Core runtime 保持通用，产品层注入 persona、workspace、memory、gateway policy | 高（同一 runtime 可服务多个产品） | 中（边界需要长期维护） | 同时建设 agent SDK 与自家 agent 产品 |
| 服务化 session app + SSE replay | FastAPI/HTTP session API + SSE replay/live fan-out + 分布式锁 | 高（Web/多用户友好） | 中高（storage/bus/lock/workspace 都要抽象） | Web 分布式 agent、多人 session、远程 workspace |

### 场景决策指南

**个人 CLI 工具 / 开发者自用** → 单渠道。多渠道是过度设计。一个入口，最快上线，最少维护。

**团队内部工具（Slack + Web 并存）** → 多渠道适配器。关键是先把 core engine 设计好，adapter 层只负责格式转换和事件映射，不含业务逻辑。注意：功能集要取所有渠道的交集，Slack 特有的 interactive buttons 在 Web 端可能无等价物，反之亦然。

**面向开发者的 API 产品** → Headless API。前端独立演化，第三方可以自由集成。代价是 API 设计必须足够稳定：一旦对外发布，破坏性变更成本极高。先把 API 设计对，再上 SDK。

**需要精美终端 UI 的开发工具** → 混合架构（参考 OpenHarness）。Python 做 agent 逻辑（生态丰富），Node.js/React/Ink 做 TUI（UI 组件丰富）。stdio JSON 协议解耦两端，TUI 甚至可以换成 Web UI 而不动后端。前提：两个进程的生命周期管理和重连逻辑必须写稳。

**同时建设 runtime SDK 和自家 agent 产品** → SDK core + product shell（OpenHarness/ohmo）。core 只放 engine、tools、memory scan、channel bus、tasks；产品壳再注入 persona、personal memory、workspace、gateway policy 和私有 skills。agent-os 也应保持这一边界：local proxy agent 和 Web distributed agent 是两种 shell，不应污染同一个 core。

**需要跨设备、跨 CLI 重启继续工作**（Claude Code Remote Session 场景）→ 考虑把远程 session 折叠为本地 task（`RemoteAgentTask` 模式），使 `--resume` 等本地能力自然适用，而不是为远程单独建一套编排路径。

**需要 Web 分布式 session / 多用户 workspace** → 服务化 session app + SSE replay（AgentScope Python 2.x / DeerFlow）。核心抽象应包括 `SessionStore`、`MessageBus`、`WorkspaceManager`、`PermissionContext`、`RunController`，而不是把 HTTP handler 直接绑到 agent loop。SSE 可先作为单向 live/replay 通道，真正需要双向实时控制时再上 WebSocket。

**需要同时接入多个 IM 平台（Slack + Telegram + Discord + 企业微信等）** → 单进程多平台网关（参考 Hermes Agent）。把所有平台差异下沉到 `BasePlatformAdapter`（三个抽象方法 + 可选 override），runner 层统一处理安全、会话、路由。关键：session key 的 DM/群组/thread 三维语义必须设计清楚——群组内是否 per-user 隔离、thread 是否共享都影响多人协作体验。

**需要 desktop/Web route 直接指向历史 session** → route resume + local snapshot + gateway rebind（Hermes Desktop）。URL 路由的 id 应是 stored session id，不一定是当前 live runtime id；刷新或 sleep/wake 后先读本地 snapshot 保持 UI 稳定，再调用 gateway `session.resume` 绑定新的 live id。否则会出现“路由还在旧 session，但 gateway runtime 已丢”的 split-brain。

**需要定时执行 agent 任务并投递到 IM 平台** → cron + 平台 live adapter 优先投递。纯 HTTP 投递无法处理 Matrix E2EE 房间（没有密钥），必须通过已建连的 live adapter 发送。同时引入文件锁防止 gateway、daemon、systemd timer 三者同时触发 tick（`fcntl` 文件锁，单机串行保证）。需要审计但不想打扰用户的任务用 `SILENT_MARKER` 标记：agent 输出本地保存，但不向平台投递。

### 常见陷阱

**多渠道没有统一权限模型**：CLI 有沙箱和工具确认机制，Slack 适配器如果直接透传命令，等于从 IM 绕过了安全层。解法：权限检查必须在 core engine 层做，不能依赖各 adapter 自己实现。Claude Code 的权限 relay 走结构化 typed notification 而非文本 regex，就是为了防止自然语言消息误触发高权限操作。

**API 没有 rate limiting，被滥用或意外超额**：Headless API 对外暴露后，没有限流等于裸奔。高并发下 LLM API 费用会在分钟级爆表。最低要求：按 API key 限制 RPM 和 TPM，超额返回 429。

**适配器层太薄，渠道特有能力被浪费**：多渠道适配器取交集意味着放弃了各渠道的差异化能力——Slack 的 Block Kit、Web 的富文本、CLI 的流式输出各有独特价值。如果产品中某个渠道是主渠道，可以为该渠道做「增强适配器」，在通用接口之上支持渠道原生能力，而不是强制所有渠道降级到最低公分母。

**stdio 协议在子进程异常退出时缺乏重连**：OpenHarness 的混合架构依赖 stdio JSON 通信，若 TUI 进程意外崩溃，后端 `BridgeSessionManager` 没有主动重连逻辑，会导致会话静默丢失。解法：后端监听子进程 `exit` 事件，自动重启或向用户暴露错误，而不是静默挂起。

**平台感知的 PII 脱敏被遗漏**：多平台场景下，同一份脱敏策略不能一刀切。Discord 的 mention 语法 `<@user_id>` 要求 LLM 持有真实 user_id，若错误脱敏则 @ 用户失效。解法：维护 `_PII_SAFE_PLATFORMS` 白名单（如 WhatsApp/Signal/Telegram），仅对这些平台做 SHA256 哈希脱敏，Discord 等依赖真实 ID 的平台明确排除。脱敏只影响 LLM 输入（system prompt），路由层始终使用原始 ID。

**Agent 实例重建导致 prompt cache 命中率骤降**：每条消息重新创建 `AIAgent` 实例，会使支持 prefix caching 的提供商（如 Anthropic）的系统提示缓存完全失效，导致 token 费用约增加 10 倍。解法：按 session key 缓存 agent 实例（`_agent_cache`），缓存失效条件改为 config 签名变化（config.yaml 改动），而非时间 TTL 或请求次数。

**Session 重置前未 flush 记忆导致上下文丢失**：daily/idle 重置触发时，若直接清空对话历史，用户之前提到的偏好、任务状态等重要信息随之消失。解法：重置前异步启动轻量 `AIAgent`，让它审阅对话历史，把重要内容写入 `MEMORY.md`/`USER.md`，flush 结果持久化到 `sessions.json`（`memory_flushed=True`），防止 gateway 重启后重复 flush。连续失败 3 次后主动放弃，避免无限重试阻塞重置流程。

**把本地 runtime id 当作 URL session id**：Web/desktop 前端如果把短生命周期 runtime id 写进路由，gateway 重启或 profile swap 后就会 404。Hermes Desktop 的做法是路由使用 stored session id，resume 后再拿 live runtime id；prompt.submit 遇到 `session not found` 时自动 `session.resume` 后重试一次。

## L2 详情

- [[channel-remote--claude-code]]
- [[channel-remote--openharness]]
- [[channel-remote--deer-flow]]
- [[channel-remote--hermes-agent]]
- [[channel-remote--agentscope]]
