---
title: "multi-agent——SimpleMem"
category: L2
parent: "[[multi-agent]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: multi-agent
created: "2026-04-15"
updated: "2026-04-15"
confidence: medium
---

## 概述

SimpleMem 的 multi-agent 支持体现在两个层面：（1）cross/ 层的 `CrossSessionOrchestrator` 作为跨 agent 会话的记忆协调器，管理多 tenant/多 session 的记忆生命周期；（2）OmniSimpleMem 的 `orchestrator.py` 编排多模态处理管道中不同模态处理器的协作。两者都不是通用 multi-agent 框架，而是为**记忆共享和协调**场景设计的专用编排器。

## 架构分析

### Cross-Session Orchestrator（cross/orchestrator.py）

`CrossSessionOrchestrator` 是 cross/ 层的核心，职责：
- **多租户会话隔离**：每个 (tenant_id, content_session_id) 对应独立的 `MemorySession`
- **记忆生命周期管理**：session_start → record_events → session_stop（finalize）→ session_end
- **跨会话记忆共享**：finalize 时提取 observations 并持久化，下次 session_start 时注入历史 context
- **并发会话支持**：threading.Lock 保护活跃 session dict，支持多 agent 并发调用

```python
class CrossSessionOrchestrator:
    _active_sessions: Dict[str, MemorySession]  # memory_session_id → session
    _lock: threading.Lock
    
    # 生命周期方法
    session_start(tenant_id, content_session_id, project, user_prompt) → HookResult
    session_message(memory_session_id, content, role) → HookResult
    session_tool_use(memory_session_id, tool_name, input, output) → HookResult
    session_stop(memory_session_id) → HookResult   # 触发 finalization
    session_end(memory_session_id) → HookResult    # 清理资源
```

### Consolidation Worker（cross/consolidation.py）

独立于主 orchestrator 的后台维护进程，执行记忆质量维护三操作：
1. **Decay**：按时间衰减旧记忆的 importance 分数（`importance *= decay_factor^n`）
2. **Merge**：合并 cosine similarity > 0.95 的近重复记忆
3. **Prune**：软删除 importance < min_threshold 的低价值记忆

可单独运行或通过 orchestrator 定期触发。

### OmniSimpleMem Orchestrator（OmniSimpleMem/omni_memory/orchestrator.py）

多模态处理编排器，协调各模态处理器（text/image/audio/video）将输入转化为统一的 `MultimodalAwarenessUnit`（MAU）。支持：
- 模态路由：`routing/router.py` 决定每类输入走哪个处理器
- 并行处理：多模态输入可并行处理后合并
- 知识演化：`evolution/` 更新 MAU 的时间权重和重要性

## 关键代码路径

### 多 Agent 共享记忆（cross/ 层典型调用序列）

```python
# Agent A 开始会话，获取历史 context
result = orchestrator.session_start(
    tenant_id="org-1",
    content_session_id="conv-abc",
    project="my-project",
    user_prompt="Help me debug the auth module"
)
# result.context_bundle → formatted past memories

# Agent A 记录事件
orchestrator.session_message(result.memory_session_id, "Found a JWT expiry bug", "assistant")
orchestrator.session_tool_use(result.memory_session_id, "bash", "git log", "...")

# 会话结束时提取 observations（跨 agent 共享的关键信息）
stop_result = orchestrator.session_stop(result.memory_session_id)
# stop_result.finalization_report.observations_count → 写入 CrossObservation 表

# Agent B 下次开始同一 project 的会话，自动获得 Agent A 的 observations
```

### Session Finalization（记忆提炼核心）

```python
# cross/session_manager.py
SessionManager.finalize_session(memory_session_id) -> FinalizationReport:
    session = _active_sessions[memory_session_id]
    
    # 1. 提取结构化观察
    observations = ObservationExtractor.extract(session.events)
    # → List[CrossObservation] (type: decision/bugfix/feature/refactor/discovery/change)
    
    # 2. 生成会话摘要
    summary = LLM.summarize(session.events)
    
    # 3. 持久化到双存储
    SQLiteStorage.save_session_record(session)
    SQLiteStorage.save_observations(observations)
    CrossSessionVectorStore.add_entries(observations_as_embeddings)
    
    return FinalizationReport(observations_count=N, summary_generated=True, entries_stored=M)
```

### 并发 Session 保护

```python
# cross/orchestrator.py
class CrossSessionOrchestrator:
    def session_start(self, ...):
        with self._lock:
            session = MemorySession(...)
            self._active_sessions[session.memory_session_id] = session
        return HookResult(context_bundle=...)
```

## 设计亮点

1. **观察类型分类（ObservationType）**：记忆不是原始文本，而是分类的结构化观察（decision/bugfix/feature/refactor/discovery/change），下次会话可按类型过滤检索，精度远高于全文检索。

2. **双存储互补**：SQLite 存结构化 session metadata 和 observations（支持精确查询），LanceDB 存 embedding 向量（支持语义检索），两者配合实现精确+模糊双通道。

3. **Soft Delete 设计**：Consolidation 的 prune 操作标记 `superseded=True` 而不物理删除，保留记忆演化的历史可审计性，可随时回滚。

4. **Duck-typed Orchestrator**：`api_mcp.py` 的 `MCPToolRegistry` 通过 `_resolve_method()` duck-typing 绑定 orchestrator，不要求实现特定接口，降低集成门槛。

## 局限性

1. **无真正的 Agent 间协商**：cross/ 层的"multi-agent"实际是多 session 的记忆共享，没有 agent 间通信、任务分发或结果聚合机制，不能替代 multi-agent 框架（如 AutoGen/CrewAI）。

2. **Consolidation 与 Orchestrator 解耦但集成弱**：`ConsolidationWorker` 需要外部调度（定时任务或手动触发），orchestrator 没有自动调度逻辑，生产部署需要额外配置。

3. **OmniSimpleMem Orchestrator 文档薄弱**：omni_memory/orchestrator.py 的跨模态编排逻辑复杂，缺少详细注释，多模态 MAU 的合并策略不透明。

4. **租户隔离仅限 cross/ 层**：核心文本版 `SimpleMemSystem` 无 tenant_id 概念，多租户场景只能通过多实例（不同 db_path）隔离，运维成本高。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
