---
title: Multi-Agent
aliases: [多智能体, multi-agent orchestration, task delegation]
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

多个 agent 协作 — 怎么拆任务、怎么分发、怎么汇总结果。

## 核心问题

- 什么时候该拆成多个 agent，什么时候单个就够？
- Agent 之间怎么通信？共享状态还是消息传递？
- 子 agent 失败了怎么处理？
- 怎么避免重复工作？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | 建立在正式任务系统之上的 agent orchestration runtime：每个子 agent 拥有独立执行环境、专属 MCP servers 和独立 transcript，通过 `AgentTool` 作为统一标准化入口被调度 | 以操作系统进程为隔离边界：每个子 agent 是独立的 `python -m openharness --headless` 子进程，以 UTF-8 文本行为通信协议；`BackgroundTaskManager` 负责进程生命周期，`SendMessageTool` 向子进程 stdin 写消息；核心逻辑约 280 行 |
| 关键特点 | 任务系统先于多智能体（子 agent 结果包装为持久化 task）；拓扑弹性（同进程/tmux 多进程/远程 backend 透明切换）；Coordinator 作为一等公民（专用 system prompt + 工具集约束） | 零依赖隔离（子进程天然隔离，无共享内存、无锁）；极简通信协议（UTF-8 文本行，任何语言可互操作）；broken pipe 检测后自动重启子进程，提升长时任务稳定性 |
| 局限 | Coordinator 模式目前是单机的，缺乏真正的分布式协调；agent 间通过 mailbox（异步写文件）通信，延迟较高 | `TeamRecord` 仅存于内存，进程重启后团队关系丢失；单向消息通信，子 agent 无法主动回调协调者；无结果合并机制，子 agent 产出仅写入日志文件 |

## 设计权衡

- **任务化 vs 临时子会话**：Claude Code 选择了任务化——子 agent 结果包装为 task，而非临时子会话，使得 async agent、状态恢复和结果查询成为可能，是架构成熟度的标志。
- **执行环境隔离 vs 状态共享**：子 agent 默认拥有独立的 MCP 连接、独立 transcript 和独立 abort controller，防止跨 agent 状态污染——隔离是默认行为，共享是显式选择。

## L2 详情

- [[multi-agent--claude-code]]
- [[multi-agent--openharness]]
