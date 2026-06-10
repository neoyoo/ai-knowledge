---
title: "Session Recovery — DeerFlow"
category: L2
parent: "[[session-recovery]]"
source: deer-flow
source_version: "v2.0-m1-rc2-11-g16391e35"
confidence: high
created: 2026-04-07
updated: 2026-06-10
---

## 概述

DeerFlow 的状态恢复仍以 LangGraph checkpointer 为底座，但最新实现已经不再是“自身零代码恢复”。Gateway/runtime 在 checkpointer 之上增加了 run lifecycle 控制层：run 启动前捕获 checkpoint snapshot，cancel 时可 interrupt + rollback，worker/manager 会清理或恢复 checkpoint，并对 orphaned inflight run 做 reconciliation。

## 架构分析

### Checkpointer 三档配置

LangGraph 接入点由 `backend/langgraph.json` 指向 `make_checkpointer()` factory；真正的后端选择在 `config.yaml` 中完成：优先兼容旧 `checkpointer:` 配置，其次使用统一 `database:` 配置，最后默认 memory。

- **InMemorySaver**：开发/测试用，进程退出即丢失
- **SqliteSaver**：单机持久化，适合本地部署
- **PostgresSaver**：多服务器部署，适合生产环境

三档配置完全由基础设施层决定，agent 代码无感知；`backend/langgraph.json` 只声明 factory 路径，不承载具体 backend 选择。

### ThreadState Schema

`ThreadState` 是 LangGraph 的 `MessagesState` 的扩展，定义了 session 的全量状态结构：

- `messages`：完整对话历史（LangChain Message 列表）
- `sandbox`：代码执行沙箱状态
- `thread_data`：中间数据，由 `ThreadDataMiddleware` 管理
- `artifacts`：agent 产出物（报告、代码文件等）
- `todos`：任务清单状态

### Per-Thread 目录系统

`ThreadDataMiddleware` 通过 `Paths` 为每个 thread 创建独立目录。base dir 由 constructor、`DEER_FLOW_HOME`、或 `{project_root}/.deer-flow` fallback 决定；新路径支持用户隔离：`{base_dir}/users/{user_id}/threads/{thread_id}/`，未传 `user_id` 时保留 legacy `{base_dir}/threads/{thread_id}/`。该目录在进程重启后保留，与 checkpointer 共同恢复文件型数据（下载文件、生成报告等）。

### 恢复机制

恢复流程：客户端发起请求时携带 `thread_id` → LangGraph 从 checkpointer 读取该 thread 的最新 checkpoint → 还原完整 `ThreadState` → runtime 按 run lifecycle 继续、取消或回滚。per-step checkpointing 能把恢复粒度收窄到最近已完成步骤，但不等于能恢复“半个工具调用内部”的进程状态；工具执行中途崩溃仍要依赖幂等工具、run rollback 或 orphaned inflight reconciliation。

### Run lifecycle rollback

DeerFlow 在 run 执行层记录 pre-run checkpoint，并把 cancel/rollback 变成 API 可控行为：

- `backend/packages/harness/deerflow/runtime/runs/worker.py:185-201` — 捕获 pre-run checkpoint snapshot
- `runtime/runs/worker.py:340-385` — cancel rollback 分支
- `runtime/runs/worker.py:456-547` — restore / delete thread checkpoint
- `runtime/runs/manager.py:466-633` — cancel、multitask 策略、orphaned run recovery
- `backend/app/gateway/routers/thread_runs.py:224-239` — HTTP cancel action
- `backend/tests/test_runtime_lifecycle_e2e.py:632-691` — E2E 验证 rollback 回到 run 前状态

这条链路对 agent-os 的启发是：session recovery 不只是“能从 checkpoint 恢复”，还要定义 run cancel 时回到哪个状态、如何处理 inflight/orphaned run，以及如何把 rollback 结果暴露给 API。

### Thread 生命周期管理

Gateway API 暴露 `DELETE /api/threads/{thread_id}` endpoint，负责清理 LangGraph checkpointer 中的 checkpoint 数据，同时删除对应的 per-thread 目录。

### 关键代码路径

- `backend/langgraph.json` — checkpointer 类型配置（memory/sqlite/postgres）
- `deerflow/agents/thread_state.py` — ThreadState schema 定义
- `backend/packages/harness/deerflow/config/paths.py` — `DEER_FLOW_HOME` / user-scoped thread dir 路径抽象
- `deerflow/agents/middlewares/thread_data_middleware.py` — per-thread 目录创建与管理
- `backend/app/gateway/routers/threads.py` — Thread CRUD API，含 DELETE 清理逻辑

## 设计亮点

- **Checkpointer + run 控制分层**：状态存储仍由 LangGraph checkpointer 承担，但运行时用 RunManager/worker 管理 cancel、rollback、orphan reconciliation
- **Per-step checkpoint**：每个已完成 step 后保存，粒度比 per-turn 细；mid-run 崩溃可从最近 checkpoint 恢复或由 run lifecycle 回滚/清理，但不承诺恢复半执行工具的内部状态
- **三档切换无缝**：开发用内存、本地用 SQLite、生产用 Postgres，切换只改配置文件
- **Postgres 支持横向扩展**：多个 Gateway 实例共享同一 Postgres，实现多服务器部署

## 局限性

- **Checkpoint 不透明**：checkpointer 是黑盒，序列化格式不可读，调试 session 状态困难
- **无人类可读导出**：不支持将 session 导出为 JSON 等可读格式（对比 OpenHarness 的文件式 session 存储）
- **存储无限增长**：checkpoint 数据默认永久保留，无自动 TTL 或清理策略
- **跨 checkpointer 迁移难**：从 SQLite 迁移到 Postgres 需要手动数据迁移，无官方工具

## 来源

- 源码版本：`v2.0-m1-rc2-11-g16391e35`
- 分析深度：源码级
