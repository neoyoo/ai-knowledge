---
title: "Session Recovery — DeerFlow"
category: L2
parent: "[[session-recovery]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 将会话持久化完全委托给 LangGraph 的 checkpointer 系统，自身零代码实现 session recovery。每个 agent 步骤（模型调用或工具执行）完成后自动保存 ThreadState，恢复时只需传入相同 `thread_id`，LangGraph 自动还原完整状态，包括消息历史、沙箱状态、thread_data、artifacts 和 todos。

## 架构分析

### Checkpointer 三档配置

LangGraph 提供三种 checkpointer，通过 `backend/langgraph.json` 配置切换：

- **InMemorySaver**：开发/测试用，进程退出即丢失
- **SqliteSaver**：单机持久化，适合本地部署
- **PostgresSaver**：多服务器部署，适合生产环境

三档配置完全由基础设施层决定，agent 代码无感知。

### ThreadState Schema

`ThreadState` 是 LangGraph 的 `MessagesState` 的扩展，定义了 session 的全量状态结构：

- `messages`：完整对话历史（LangChain Message 列表）
- `sandbox`：代码执行沙箱状态
- `thread_data`：中间数据，由 `ThreadDataMiddleware` 管理
- `artifacts`：agent 产出物（报告、代码文件等）
- `todos`：任务清单状态

### Per-Thread 目录系统

`ThreadDataMiddleware` 为每个 thread 创建独立目录 `backend/.deer-flow/threads/{thread_id}/`，用于存储文件型数据（下载文件、生成报告等）。该目录在进程重启后保留，与 checkpointer 共同实现完整的 session 恢复。

### 恢复机制

恢复流程：客户端发起请求时携带 `thread_id` → LangGraph 从 checkpointer 读取该 thread 的最新 checkpoint → 还原完整 `ThreadState` → agent 从上次中断点继续执行。per-step checkpointing 意味着即使工具调用执行到一半，重启后也能从该步骤之后继续，而非重跑整个 turn。

### Thread 生命周期管理

Gateway API 暴露 `DELETE /api/threads/{thread_id}` endpoint，负责清理 LangGraph checkpointer 中的 checkpoint 数据，同时删除对应的 per-thread 目录。

### 关键代码路径

- `backend/langgraph.json` — checkpointer 类型配置（memory/sqlite/postgres）
- `deerflow/agents/thread_state.py` — ThreadState schema 定义
- `deerflow/agents/middlewares/thread_data_middleware.py` — per-thread 目录创建与管理
- `backend/app/gateway/routers/threads.py` — Thread CRUD API，含 DELETE 清理逻辑

## 设计亮点

- **零代码持久化**：agent 逻辑完全不涉及存储细节，checkpointer 由 LangGraph 基础设施透明处理
- **Per-step checkpoint**：每个 tool call / model call 完成后立即保存，粒度比 per-turn 细，mid-execution 崩溃可恢复
- **三档切换无缝**：开发用内存、本地用 SQLite、生产用 Postgres，切换只改配置文件
- **Postgres 支持横向扩展**：多个 Gateway 实例共享同一 Postgres，实现多服务器部署

## 局限性

- **Checkpoint 不透明**：checkpointer 是黑盒，序列化格式不可读，调试 session 状态困难
- **无人类可读导出**：不支持将 session 导出为 JSON 等可读格式（对比 OpenHarness 的文件式 session 存储）
- **存储无限增长**：checkpoint 数据默认永久保留，无自动 TTL 或清理策略
- **跨 checkpointer 迁移难**：从 SQLite 迁移到 Postgres 需要手动数据迁移，无官方工具

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
