---
title: Channel & Remote
aliases: [渠道, remote execution, multi-channel]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[runtime-state]]"
    type: uses
sources: [claude-code, openharness]
---

## 一句话定义

多渠道接入（CLI、Web、Slack、IDE）与远程执行能力。

## 核心问题

- 不同渠道的输入输出格式差异怎么抹平？
- 远程执行和本地执行的安全模型有什么区别？
- 渠道间的会话状态怎么同步？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | Channel（MCP 子协议，解决外部消息通道异步接入）+ Remote Session（把云端 agent 实例折叠回本地 task 系统统一编排），两者共同将 agent 从终端内对话循环扩展为跨设备可恢复 agent runtime | Python 后端 + React/Ink TUI 前端通过 stdio JSON 协议通信的混合架构；`BridgeSessionManager` 管理长生命周期子进程；cron 系统通过 `RemoteTriggerTool` 支持定时触发 agent 执行；当前"远程"本质上是本地子进程管理 |
| 关键特点 | Channel 复用整套 MCP 基础设施（无需单独发明 IM 插件 runtime）；权限 relay 走结构化 typed notification 而非文本 regex（防止自然语言误触发）；`RemoteAgentTask` 把远程 session 折叠为本地 task，支持 `--resume` 跨 CLI 重启 | 混合语言架构（Python + Node.js React/Ink），stdio JSON 协议解耦，各语言专注擅长领域；cron 调度内置，JSON 存储简单可审计；`WorkSecret` 凭证编码为未来网络化扩展预留接口形态 |
| 局限 | Channel 依赖 Claude.ai OAuth，纯 API key 用户无法使用；Remote session 要求 git repo + git remote；远程 session 只能 HTTP 轮询不能 WebSocket 推送 | 无 WebSocket 服务端，无 OAuth，无云端 channel；所谓"远程"是本地子进程管理；`WorkSecret` 机制已设计但未连接任何活跃网络端点；stdio JSON 通信在进程异常退出时缺乏健壮的重连机制 |

## 设计权衡

- **Channel = MCP 子协议 vs 独立插件体系**：Claude Code 选择了在 MCP 协议上构建 Channel，复用连接管理、auth、transport 和 plugin 体系，没有单独发明 IM/手机插件 runtime——工程复用度极高，但功能边界受限于 MCP 能力范围。
- **RemoteAgentTask 折叠 vs 远程专用编排路径**：把远程 session 包装成本地 task，与多智能体系统共享同一编排抽象，使 `--resume` 等本地能力自然适用于远程 agent——统一视图的代价是需要维护本地-远程状态映射层。

## L2 详情

- [[channel-remote--claude-code]]
- [[channel-remote--openharness]]
