---
title: "runtime-state——SimpleMem"
category: L2
parent: "[[runtime-state]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: runtime-state
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

SimpleMem 的运行时状态管理分两层：（1）in-process 状态——`SimpleMemSystem` 持有 `MemoryBuilder.dialogue_buffer` 和 `previous_entries` 作为会话内临时状态；（2）持久化状态——`cross/` 层通过 SQLite（结构化 metadata）+ LanceDB（向量 embeddings）双存储实现跨进程持久状态，数据模型通过 `cross/types.py` 的 Pydantic 模型统一。

## 架构分析

### In-Process 运行时状态（文本版）

`MemoryBuilder` 维护两个运行时状态变量：
```python
self.dialogue_buffer: List[Dialogue] = []    # 待处理对话窗口缓冲
self.previous_entries: List[MemoryEntry] = []  # 上一窗口记忆（用于去重 context）
self.processed_count: int = 0                # 已处理对话数计数
```

`SimpleMemSystem` 持有所有核心组件引用（无状态组合），生命周期与进程相同。

### 持久化状态层（cross/）

双存储架构：

| 存储 | 文件 | 数据类型 | 查询方式 |
|------|------|---------|---------|
| SQLite | `cross/storage_sqlite.py` | SessionRecord, SessionEvent, CrossObservation, SessionSummary | SQL 精确查询 |
| LanceDB | `cross/storage_lancedb.py` | CrossObservation embeddings | 向量语义搜索 |

### 核心状态类型（cross/types.py）

```python
# 会话记录（SQLite）
class SessionRecord(BaseModel):
    memory_session_id: str
    tenant_id: str
    content_session_id: str
    project: str
    status: SessionStatus        # pending / active / stopped / ended
    created_at: datetime
    stopped_at: Optional[datetime]

# 事件（SQLite）
class SessionEvent(BaseModel):
    memory_session_id: str
    kind: EventKind              # message / tool_use / file_change / note / system
    title: Optional[str]
    payload_json: Optional[str]
    redaction_level: RedactionLevel  # none / partial / full
    timestamp: datetime

# 跨会话观察（SQLite + LanceDB）
class CrossObservation(BaseModel):
    memory_session_id: str
    type: ObservationType        # decision/bugfix/feature/refactor/discovery/change
    title: str
    narrative: Optional[str]
    timestamp: datetime
    importance: float            # 用于 consolidation decay/prune

# 会话摘要（SQLite）
class SessionSummary(BaseModel):
    memory_session_id: str
    text: str
    created_at: datetime

# Context Bundle（runtime，不持久化）
class ContextBundle(BaseModel):
    session_summaries: List[SessionSummary]
    observations: List[CrossObservation]
    semantic_matches: List[str]
    total_tokens: int
    formatted_context: str

# 结束化报告（runtime，不持久化）
class FinalizationReport(BaseModel):
    observations_count: int
    summary_generated: bool
    entries_stored: int
```

### 活跃 Session 状态管理

```python
# cross/orchestrator.py
class CrossSessionOrchestrator:
    _active_sessions: Dict[str, MemorySession]  # 内存中的活跃 session
    _lock: threading.Lock                        # 并发保护
```

`MemorySession` 是 runtime 对象，包含：
- `memory_session_id`、`tenant_id`、`project` 等元数据
- `EventCollector` 实例（事件缓冲）
- `status: SessionStatus`（状态机）

## 关键代码路径

### Session 状态机转换

```python
# SessionStatus 状态转换序列:
# pending → active (session_start)
# active → stopped (session_stop, 触发 finalization)
# stopped → ended (session_end, 清理资源)

# cross/session_manager.py
def session_start(...) -> HookResult:
    session = MemorySession(status=SessionStatus.active, ...)
    _active_sessions[session.memory_session_id] = session
    bundle = context_injector.build_context_bundle(...)
    return HookResult(context_bundle=bundle)

def finalize_session(memory_session_id) -> FinalizationReport:
    session = _active_sessions[memory_session_id]
    session.status = SessionStatus.stopped
    events = session.event_collector.flush()    # 清空 buffer
    storage.save_events(events)
    observations = obs_extractor.extract(events)
    storage.save_observations(observations)
    vector_store.add_observations(observations)
    return FinalizationReport(...)

def end_session(memory_session_id):
    session = _active_sessions.pop(memory_session_id)  # 从内存移除
    session.status = SessionStatus.ended
```

### SQLite 持久化（storage_sqlite.py）

```python
class SQLiteStorage:
    def __init__(self, db_path: str):
        self._conn = sqlite3.connect(db_path, check_same_thread=False)
        self._setup_tables()  # 建表（sessions/events/observations/summaries）
    
    def save_session_record(self, session: SessionRecord):
        self._conn.execute(
            "INSERT INTO sessions VALUES (?, ?, ?, ?, ?, ?)",
            (session.memory_session_id, session.tenant_id, ...)
        )
    
    def get_recent_summaries(self, tenant_id: str, project: str, limit: int) -> List[SessionSummary]:
        # 按 created_at DESC 查询，支持 tenant_id + project 过滤
```

### LanceDB 向量存储（storage_lancedb.py）

```python
class CrossSessionVectorStore:
    def __init__(self, db_path: str, embedding_model: EmbeddingModel):
        self._db = lancedb.connect(db_path)
        self._table = self._db.open_table("observations")
    
    def add_observations(self, observations: List[CrossObservation]):
        # 生成 embedding，存入 LanceDB
        vectors = [embedding_model.encode(obs.narrative or obs.title) for obs in observations]
        self._table.add([{"vector": v, **obs.dict()} for obs, v in zip(observations, vectors)])
    
    def search(self, query: str, top_k: int) -> List[CrossObservation]:
        vector = embedding_model.encode(query)
        results = self._table.search(vector).limit(top_k).to_pandas()
        return [CrossObservation(**r) for _, r in results.iterrows()]
```

## 设计亮点

1. **双存储互补设计**：SQLite 处理结构化精确查询（session_id、tenant_id、time range），LanceDB 处理模糊语义搜索，两者各司其职，不用一个存储勉强两用。

2. **SessionStatus 状态机**：明确的 `pending→active→stopped→ended` 状态机，防止乱序调用（如未 start 就 stop），状态转换可被外部观测用于监控。

3. **EventCollector 写后缓冲**：事件先在内存缓冲，session_stop 时批量 flush，避免每条消息都触发 SQLite 写入（I/O 放大），对长会话性能影响显著。

4. **ContextBundle runtime 不持久化**：每次 session_start 动态计算 ContextBundle，不缓存到磁盘，确保 context 始终反映最新记忆状态，不会读到过期缓存。

5. **importance 字段为 consolidation 铺路**：`CrossObservation.importance` 在写入时默认为 1.0，ConsolidationWorker 定期按时间衰减，实现记忆重要性随时间自然降低。

## 局限性

1. **SQLite 单连接并发风险**：`check_same_thread=False` 允许多线程共享连接，但 SQLite 的写锁是全库级，高并发写入（多 tenant 同时 stop session）会有锁竞争，不适合高吞吐场景。

2. **in-process 状态不可分布式**：`_active_sessions` dict 在内存中，进程重启即丢失所有进行中的 session 状态（已 stop 的会持久化，但 active 的不会），不适合有进程重启需求的部署。

3. **LanceDB 无分区**：所有 tenant 的 observations 在同一个 LanceDB table，随数据增长向量搜索的 recall 可能下降（需 tenant_id 后过滤），全量扫描成本线性增长。

4. **MemoryBuilder 状态无序列化**：`dialogue_buffer` 是纯内存状态，进程崩溃时缓冲中未处理的对话丢失，缺乏 WAL（Write-Ahead Log）或 checkpoint 机制。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
