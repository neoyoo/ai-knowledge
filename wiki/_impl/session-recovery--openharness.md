---
title: "Session Recovery — OpenHarness"
category: L2
parent: "[[session-recovery]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness 的 session recovery 层以纯 JSON 文件为存储介质，在每个用户 turn 结束后自动快照完整会话状态，支持通过 `--continue`（恢复最近）和 `--resume`（恢复指定）两个 CLI 标志重入会话。整个实现仅 178 行，设计极简但功能完整。

## 架构分析

### 快照写入机制

每次用户 turn 结束后调用 `save_session_snapshot()`，将会话状态序列化为 JSON 写入 `~/.openharness/sessions/{project-name}-{sha1}/session-{id}.json`，同时维护一份 `latest.json` 副本（非符号链接，直接拷贝）。快照内容包含：`session_id`、`cwd`、`model`、`system_prompt`、完整 `messages` 列表（Pydantic 序列化）、累计 usage 统计、`created_at` 时间戳、以及取自首条用户消息的 80 字符摘要。

### 快照列举与加载

`list_session_snapshots()` 通过 glob 匹配 `session-*.json`，按文件 mtime 排序返回。`--continue` 标志在启动时加载 `latest.json`，`--resume` 标志接收具体 session id 并加载对应文件。两条路径均在 `cli.py` 的参数解析阶段处理，早于 REPL 主循环启动。

### 导出功能

`export_session_markdown()` 将会话转换为人类可读的 Markdown 格式文本，供归档或分享使用。该函数独立于恢复流程，按需调用。

### 关键代码路径

- `openharness/services/session_storage.py` — `save_session_snapshot()` / `list_session_snapshots()` / `export_session_markdown()` 全部实现，178 行
- `openharness/config/paths.py` — 定义 `~/.openharness/sessions/` 路径常量及 project-name+sha1 目录命名规则
- `openharness/cli.py` — `--continue` / `--resume` 标志解析与会话加载入口

## 设计亮点

- **人类可读存储**：纯 JSON 文件，无 SQLite、无二进制格式，可直接用文本编辑器查看和手动修复
- **极致精简**：完整 session recovery 层仅 178 行代码，无外部依赖
- **天然可移植**：JSON 文件可直接拷贝跨机器迁移，或纳入版本控制
- **80 字符摘要索引**：每个 snapshot 自带人类可读摘要，`list` 命令输出友好

## 局限性

- **Session ID 不稳定**：每次 REPL 调用生成新 uuid，无法在整个项目生命周期内保持稳定标识（对比 Claude Code 的持久 session id）
- **仅 turn 间快照**：快照点在 turn 结束后写入，turn 执行中途崩溃无法恢复
- **无命名会话**：session 只能通过 id 或"最近"定位，不支持用户自定义名称
- **元数据贫乏**：缺少标签、分支关联、工具调用统计等丰富元数据，难以支持复杂会话管理场景

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
