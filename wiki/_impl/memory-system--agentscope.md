---
title: "memory-system——agentscope"
category: L2
parent: "[[memory-system]]"
source: agentscope
source_version: "v2.0.1-11-g0e5418e8"
concept: "memory-system"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 当前源码不再提供旧版 `memory/`、`long_term_memory`、`mem0`、`ReMe` 或 `rag/` 子系统。它的“记忆”能力主要表现为 **session-local runtime memory**：

- `AgentState.context` 保存未压缩对话上下文
- `AgentState.summary` 保存压缩摘要
- workspace/offloader 保存被压缩 context 或大工具结果的 evidence
- `StorageBase` 持久化 session state 和 message history
- skills/MCP/workspace 为 agent 提供外部扩展能力，但不是跨会话长期记忆系统

因此，AgentScope 2.x 不应再被列为“长期记忆系统”代表源；更适合作为 session state、context compression 和 workspace offload 的参考。

---

## 当前源码事实

### Session-local memory

`AgentState` 中与 memory 相关的字段：

| 字段 | 作用 |
|---|---|
| `summary` | 被压缩历史摘要，下次模型输入时作为上下文投影 |
| `context` | 未压缩 `Msg` 列表 |
| `tool_context.read_file_cache` | 文件工具的读取 cache，带 mtime 校验 |
| `tasks_context.tasks` | 当前任务状态 |

这些状态随 `SessionRecord.state` 持久化，恢复后继续服务同一个 session。

### Context offload

当 `compress_context()` 触发时，被压缩的原始 messages 可通过 `offloader.offload_context(session_id, msgs)` 落到 workspace。摘要中会附加 `<system-reminder>`，提示 agent 可按路径回查。

当单次 tool result 超过 `ContextConfig.tool_result_limit` 时，工具结果也可被截短并 offload。

### Message history

`ChatService` 在每轮运行中把 user message 和最终 assistant reply message 写入 storage。message history 是 session recovery 和 UI 展示的数据源，但不是带检索/更新策略的长期记忆系统。

---

## 与旧版页的差异

旧页中以下结论已不适用于当前 Python 2.x：

- Working Memory `MemoryBase`
- Redis/SQLAlchemy/Tablestore memory backend
- Long-Term Memory `mem0` / `ReMe`
- `record_to_memory` / `retrieve_from_memory`
- mark-based memory partition
- `StateModule` 统一序列化
- `rag/KnowledgeBase` 与 VDB store

这些路径在当前 `src/agentscope/` 中未找到。

---

## 设计亮点

### 1. 把 session-local memory 做进 AgentState

AgentScope 2.x 没有把 memory 做成单独插件，而是把 summary/context/tool cache 放进 `AgentState`。这让 Web runtime 的恢复路径简单：读 `SessionRecord.state` 即可恢复当前对话工作现场。

### 2. Offload 比长期记忆更贴近代码代理

对本地/代码 agent，很多“记忆”其实是大工具结果和文件证据。AgentScope 2.x 的 workspace offload 直接把证据落到 agent 可访问路径，比强行写入向量记忆更适合调试和回查。

### 3. 长期记忆留给外部系统

当前架构通过 MCP/skills/workspace 暴露外部能力，因此长期记忆可以作为独立 MCP server 或 app storage extension 接入，而不是内置到 agent core。

---

## 局限性

### 1. 缺少跨会话长期记忆抽象

没有“什么值得记、怎么写入、怎么召回、怎么过期”的长期 memory pipeline。需要用户偏好、跨 session recall 或个人知识库时，需要外接 MemPalace、TencentDB Agent Memory、mem0、Zep 等系统。

### 2. Summary 是压缩上下文，不是事实记忆

`state.summary` 服务当前 session 续跑，目标是保持任务连续性。它不适合当作长期用户画像或可检索知识库。

### 3. Offload 缺少检索层

offloaded context/tool result 有路径回链，但没有内置索引、语义检索、过期策略或 evidence ranking。长任务中 evidence 数量增长后，需要外层 index。

### 4. Message history 不等于 memory

storage 中的 messages 可恢复和展示，但没有自动抽取、去重、冲突解决、召回预算等 memory lifecycle 机制。

---

## 对 agent-os 的借鉴

agent-os 应把 memory 分成三层 ABC：

- `SessionMemory`：当前 session 的 summary/context/tool cache，跟 `AgentState` 走
- `EvidenceStore`：大工具结果和文件证据，支持 local file / object store / DB
- `LongTermMemory`：跨会话记忆，支持 recall/write/update/expire/evaluate

AgentScope 2.x 主要参考 `SessionMemory + EvidenceStore`；TencentDB Agent Memory / MemPalace 更适合参考 `LongTermMemory` 和 evidence-backed recall。

---

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
- 核心文件：
  - `src/agentscope/state/_state.py`
  - `src/agentscope/agent/_agent.py`
  - `src/agentscope/agent/_config.py`
  - `src/agentscope/app/storage/_base.py`
  - `src/agentscope/app/storage/_model/_session.py`
  - `src/agentscope/app/_service/_chat.py`
  - `src/agentscope/workspace/_offload_protocol.py`
  - `src/agentscope/workspace/_local_workspace.py`
