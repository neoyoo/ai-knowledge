---
title: "Multi-Agent — OpenHarness"
category: L2
parent: "[[multi-agent]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness 的多智能体系统通过操作系统进程作为隔离边界，每个子 agent 是一个独立的 `python -m openharness --headless` 子进程，以 UTF-8 文本行为通信协议。主模型调用 `agent` 工具触发派生，`SendMessageTool` 向运行中的子进程 stdin 写入消息，`BackgroundTaskManager` 负责进程生命周期管理。整套实现约 280 行，无共享内存、无锁、无复杂序列化。

## 架构分析

### 进程派生模型

模型通过 `agent` 工具发起多智能体调用，参数包括 `prompt`（任务描述）、`description`（展示用说明）和可选的 `mode`（`local_agent` / `remote_agent` / `in_process_teammate`，当前仅 subprocess 路径有效）。`BackgroundTaskManager.create_agent_task()` 构造完整的 CLI 命令，将 prompt 写入子进程 stdin，将 stdout 重定向到日志文件，并在 `dict[str, TaskRecord]` 中记录任务状态。

### 任务与团队注册

每个任务由唯一 ID 索引，`TaskRecord` 存储进程句柄、日志路径和状态。团队（Team）是纯内存 `dict[str, TeamRecord]`，将团队名称映射到任务 ID 列表，仅在当前会话生命周期内有效。

### 消息传递协议

`SendMessageTool` 通过向目标子进程的 stdin 追加文本行实现消息传递，这是一种极简的单向 push 协议。子进程管道断裂时触发自动重启逻辑，确保长时任务的基本鲁棒性。

### 关键代码路径

- `tasks/manager.py` — `BackgroundTaskManager`：进程派生、任务状态追踪、自动重启
- `coordinator/coordinator_mode.py` — 协调者模式入口，判断任务分发策略
- `coordinator/agent_definitions.py` — agent 工具的参数 schema 定义
- `tools/agent_tool.py` — `agent` 工具实现，调用 `create_agent_task()`
- `tools/send_message_tool.py` — `SendMessageTool`，向子进程 stdin 写消息

## 设计亮点

- **零依赖隔离**：子进程天然隔离，无需共享内存、锁或复杂序列化，系统边界清晰
- **极简通信协议**：以 UTF-8 文本行为消息格式，任何语言的 agent 均可互操作
- **轻量实现**：核心逻辑 ~280 行，管理器 + 协调者 + 工具定义全部包含在内
- **自动重启**：broken pipe 检测后自动重启子进程，提升长时任务稳定性

## 局限性

- **无真正的 in-process 模式**：`in_process_teammate` 模式名存实亡，实际仍走 subprocess 路径，无法实现真正的内存共享
- **团队注册非持久化**：`TeamRecord` 仅存于内存，进程重启后团队关系丢失，无法跨会话恢复
- **单向消息通信**：stdin/stdout 管道只支持单向推送，子 agent 无法主动回调协调者，缺乏双向结构化通信
- **无结果合并机制**：子 agent 产出仅写入日志文件，主 agent 需自行解析，无内置结果聚合或冲突消解
- **无上下文传递**：协调者不向子 agent 传递结构化上下文（如 session state、tool registry），子 agent 完全从 prompt 文本中理解任务

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
