---
title: "Channel & Remote — Claude Code"
category: L2
parent: "[[channel-remote]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 通过 Channel 和 Remote Session 两个机制，把自身从"终端内对话循环"扩展成"跨设备、跨环境的可恢复 agent runtime"。Channel 解决外部消息通道（手机、IM、IDE）异步接入问题，本质是 MCP 协议上的特化子协议；Remote Session 解决 agent 在云端持续执行问题，把远程运行实例通过 `RemoteAgentTask` 重新折叠回本地 task 系统统一编排。

## 架构分析

### Channel：MCP 上的子协议

Channel 不是独立插件体系，而是 MCP 协议的 capability profile。

**协议层**（`channelNotification.ts`）定义了三类通知：
- `notifications/claude/channel` — 入站消息（外部推送到当前 agent session）
- `notifications/claude/channel/permission` — 结构化权限回复（外部批准/拒绝 tool use）
- `notifications/claude/channel/permission_request` — 出站权限请求（Claude Code 向 channel server 发送）

设计要点：Channel server 本质上仍是一个 MCP server，出站消息走普通 MCP tools，入站走 `notification` 机制。这样复用了 MCP 的连接管理、transport 抽象、auth/reconnect、capabilities 谈判，无需单独发明插件 runtime。

**运行时接线**（`useManageMCPConnections.ts`）：
1. MCP server 连接成功后，调用 `gateChannelServer()` 判断该连接是否允许成为 channel
2. 通过验证的连接注册 notification handler
3. 收到消息后包装成 `<channel>` 消息并 `enqueue()` 进主消息队列

Channel 消息和本地用户输入、bridge 通知共享同一条 prompt 消费路径，无特殊调度。

### Channel 消息包装

`wrapChannelMessage()` 把外部消息包装为结构化 XML：

```
<channel source="serverName" meta_key1="val1" ...>
  外部消息内容
</channel>
```

`source` 记录来源 serverName，`meta` 字段转为 XML attributes。`SAFE_META_KEY` 正则限制 attribute 名称，阻止恶意 key 破坏 XML 结构（prompt injection 防御）。

### 分层 Gate 机制

Channel 采用"默认关闭 + 双重显式启用"安全态度。`gateChannelServer()` 依次检查：

1. **Capability gate**：server 必须声明 `experimental['claude/channel']`
2. **Feature gate**：GrowthBook `tengu_harbor` 总开关
3. **Auth gate**：必须是 Claude.ai OAuth，拒绝纯 API key
4. **Policy gate**：team/enterprise 需要 org policy 显式开启
5. **Session gate**：当前 session 必须通过 `--channels` 声明允许的 channel
6. **Trust gate**：plugin 必须通过 allowlist 或 marketplace 校验

`channelAllowlist.ts` 的 allowlist 粒度是 plugin 级别而非 server 级别，原因是工程化取舍：如果 plugin 已被攻陷，再做 per-server 控制不能增加实质能力边界——信任边界与安装边界一致。

`--channels` 参数（`bootstrap/state.ts` 解析为 `allowedChannels`）区分 `plugin` entry 和 `server` entry，plugin entry 需 marketplace 验证，server entry 默认不通过 allowlist（除非 dev bypass）。

### 权限 Relay

Channel 权限 relay 是这组代码中设计最精细的部分。

**问题**：当 Claude Code 弹出本地权限确认时，如何允许用户通过手机/IM 远程批准？

**方案**（`channelPermissions.ts` + `interactiveHandler.ts`）：
1. 权限请求产生 `toolUseID`
2. `shortRequestId()` 生成 5 位字母 ID（排除 `l` 避免和 `1/I` 混淆，有 substring blocklist 过滤脏词）
3. `interactiveHandler.ts` 向所有已连接 channel 发送 `permission_request` notification
4. Channel server 把人类回复解析为 `{request_id, behavior}` 结构
5. Claude Code 只接受特定 `notifications/claude/channel/permission`，不从普通文本 regex 拦截 "yes xxx"
6. `createChannelPermissionCallbacks()` 用 pending map 匹配请求和回复

关键设计：权限批准脱离普通 chat 文本，完全走结构化 notification，极大降低自然语言误触发权限确认的风险。

信任模型：channel server 被攻陷后理论上可伪造 permission reply，但作者认为该 server 本来就已具备持续 conversation injection 能力——"伪造批准"是更快路径，不是本质上更强的新能力。因此安全边界在 allowlist，而非 per-notification 加密签名。

### Remote Session 架构

远程 agent 不是"特殊模式"，而是 claude.ai / CCR 上的真实 session 资源，本地 CLI 只是控制端和观察端。

**前置资格检查**（`remoteSession.ts` 的 `checkBackgroundRemoteSessionEligibility()`）：
- org policy 允许远程会话
- 已登录 Claude.ai（OAuth）
- 有 remote environment
- 处于 git 仓库
- 有 git remote / GitHub app，或可走 git bundle seed 路径

多项前置约束在创建前收敛，防止"直接发请求试试看"的失败模式。

**远程 Session 创建与恢复**（`teleport.tsx`）：

代码来源决策树（优先级从高到低）：
1. GitHub source（标准 GitHub clone）
2. git bundle seed（只有本地 `.git` 的仓库）
3. 空 sandbox（兜底）

`teleport.tsx` 还负责：session context / initial events / permission mode 设置、远程事件轮询、teleported session 恢复、session 归档。

**Sessions API**（`teleport/api.ts`）：

远程会话被正规化为 RESTful 资源，而非 ad-hoc RPC：
- `fetchSession()` — 获取 session 状态
- `sendEventToRemoteSession()` — 向远程 session 写入 user event
- `updateSessionTitle()` — 更新 session 标题
- `GET /v1/sessions/{id}/events` — 轮询事件流

`teleportFromSessionsAPI()` 优先走 `teleport-events` endpoint，失败 fallback 到 `session-ingress`——两条路的存在说明架构有过渡期考量。

### RemoteAgentTask：把远程 Session 折叠回本地

`RemoteAgentTask.tsx` 把远程 session 包装成本地 task，统一进入 task orchestration 层：

- 注册 task state
- 持久化 sidecar metadata，支持 `--resume` 跨 CLI 重启
- 周期性轮询远程 event log
- 从远程日志提取 todo / ultraplan / remote review 结果
- 完成/失败/kill 时触发本地 notification

这与多智能体架构结论一致：Claude Code 的"子 agent / 远程 agent"都被 task 化，task 是稳定的 orchestration 抽象。

### 两层传输体系

| 传输层 | 实现 | 用途 |
|------|------|------|
| MCP WebSocket | `mcpWebSocketTransport.ts` | channel server、外部工具服务 |
| Sessions API HTTP | `teleport/api.ts` | 远程 Claude 实例的创建/轮询/控制 |

`mcpWebSocketTransport.ts` 兼容 Node `ws` 和 Bun 原生 WebSocket，将收发内容统一为 JSON-RPC message。两层传输最终都汇入同一 runtime 编排层。

### 关键代码路径
- `src/services/mcp/channelNotification.ts` — Channel 协议定义（通知类型、消息包装）
- `src/services/mcp/channelAllowlist.ts` — Plugin 级别 allowlist
- `src/services/mcp/channelPermissions.ts` — 权限 relay 机制
- `src/hooks/toolPermission/handlers/interactiveHandler.ts` — 权限请求分发
- `src/services/mcp/useManageMCPConnections.ts` — Channel 运行时接线
- `src/utils/teleport/api.ts` — Sessions API（远程 session CRUD）
- `src/utils/teleport.tsx` — 远程 session 创建、轮询、恢复
- `src/tasks/RemoteAgentTask/RemoteAgentTask.tsx` — 远程 session → 本地 task 适配器
- `src/utils/background/remote/remoteSession.ts` — 远程 session 资格检查
- `src/utils/mcpWebSocketTransport.ts` — MCP WebSocket transport 实现

## 设计亮点

- **Channel = MCP 子协议**：复用整套 MCP 基础设施（连接管理、auth、transport、plugin 体系），没有单独发明 IM/手机插件 runtime，工程复用度极高
- **权限 relay 的结构化事件设计**：权限批准走 typed notification 而非文本 regex，从协议层杜绝自然语言误触发，短 ID 设计兼顾算法唯一性和手机端人类输入体验（排除 `l`、过滤脏词）
- **RemoteAgentTask 折叠抽象**：把"远程云端运行的 session"映射回本地 task 系统，统一编排视图——无论 agent 在哪里运行，本地都能 `--resume`
- **多角色执行位点统一**：本地进程内 agent、本地 task 系统后台 agent、远程 environment agent 三类执行位点，通过 task 抽象统一管理
- **代码来源决策树**：GitHub clone → git bundle seed → 空 sandbox 的三级降级，保证 remote session 在各种仓库条件下都能启动

## 局限性

- **Channel 依赖 Claude.ai OAuth**：纯 API key 用户无法使用 Channel 功能，功能与认证方式强绑定
- **Remote session 要求多个前提条件**：git repo + git remote / GitHub app，本地-only 项目无法使用远程模式
- **远程 session 只能轮询不能推送**：`pollRemoteSessionEvents()` 是 HTTP 轮询，缺乏 WebSocket 推送，增加延迟和服务端负载
- **Channel allowlist 信任粒度粗**：plugin 级别 allowlist 意味着一个 plugin 内所有 channel server 共享信任边界，单个 server 妥协会影响整个 plugin 下的所有 server
- **`teleportFromSessionsAPI()` 双路回退未来维护成本**：`teleport-events` 和 `session-ingress` 两条路并存，是过渡期设计，长期需要清理

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
