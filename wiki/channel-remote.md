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
sources: [claude-code]
---

## 一句话定义

多渠道接入（CLI、Web、Slack、IDE）与远程执行能力。

## 核心问题

- 不同渠道的输入输出格式差异怎么抹平？
- 远程执行和本地执行的安全模型有什么区别？
- 渠道间的会话状态怎么同步？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | Channel（MCP 子协议，解决外部消息通道异步接入）+ Remote Session（把云端 agent 实例折叠回本地 task 系统统一编排），两者共同将 agent 从终端内对话循环扩展为跨设备可恢复 agent runtime |
| 关键特点 | Channel 复用整套 MCP 基础设施（无需单独发明 IM 插件 runtime）；权限 relay 走结构化 typed notification 而非文本 regex（防止自然语言误触发）；`RemoteAgentTask` 把远程 session 折叠为本地 task，支持 `--resume` 跨 CLI 重启 |
| 局限 | Channel 依赖 Claude.ai OAuth，纯 API key 用户无法使用；Remote session 要求 git repo + git remote；远程 session 只能 HTTP 轮询不能 WebSocket 推送 |

## 设计权衡

- **Channel = MCP 子协议 vs 独立插件体系**：Claude Code 选择了在 MCP 协议上构建 Channel，复用连接管理、auth、transport 和 plugin 体系，没有单独发明 IM/手机插件 runtime——工程复用度极高，但功能边界受限于 MCP 能力范围。
- **RemoteAgentTask 折叠 vs 远程专用编排路径**：把远程 session 包装成本地 task，与多智能体系统共享同一编排抽象，使 `--resume` 等本地能力自然适用于远程 agent——统一视图的代价是需要维护本地-远程状态映射层。

## L2 详情

- [[channel-remote--claude-code]]
