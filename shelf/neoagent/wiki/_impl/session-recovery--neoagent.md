---
title: "session-recovery — neoagent"
category: L2
parent: "[[session-recovery]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: session-recovery
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的 session 恢复系统由 `Session`/`SessionState` 双数据类 + `SessionStorage` Protocol + `JsonFileStorage` 默认实现组成（`session.py` 258 行）。关键设计：**per-turn auto-save**（`save_if_storage()` 在 `QueryLoop` 每轮 end_turn 和 tool_use 分支末尾各调一次，无配置时 no-op）；**原子 fork**（`Session.fork()` 深拷贝 messages+state+metadata 生成新 session 用于 A/B 分叉）；**resume validation with warnings**（`NeoAgent.resume()` 可选校验 workspace_path 存在性 + session updated_at 是否 stale，通过 `SessionResumeWarningEvent` 告警而不抛错）。文件权限 0o600 + 路径逃逸检查双层保证 JSON 持久化安全。

## 架构分析

### 数据层分层

```python
@dataclass
class FreedToolResult:
    id: str; tool_name: str; size: int
    preview: str; original_content: str

@dataclass
class SessionState:
    previous_summary: str | None = None
    compression_failures: int = 0
    memory_tool_calls: int = 0
    memory_token_baseline: int = 0
    total_input_tokens: int = 0
    total_output_tokens: int = 0
    promoted_tools: set[str]                       # MCP 提升过的工具名
    freed_tool_results: dict[str, FreedToolResult] # 可恢复的工具结果
    recalled_this_turn: set[str]                   # 本轮 recall 的 id（不持久化？见下）
    tool_use_to_tool_name: dict[str, str]          # auto_free_after 查表
    tool_result_ages: dict[str, int]               # age 累计

@dataclass
class Session:
    id: str
    messages: list[Message]
    state: SessionState
    created_at: datetime
    updated_at: datetime
    metadata: dict
    # _storage 通过 __post_init__ 里 object.__setattr__ 手动设置（避开 dataclass 字段）
```

`SessionState` 里 **11 个字段全部参与持久化**（`_session_to_dict` / `_session_from_dict`）——包括 `freed_tool_results` 这个可能很大的字典，意味着 JSON 文件可能数 MB。

### 持久化流程

- **`Session.save_if_storage()`**：从 `_storage` attr 读 binding，若有则 `updated_at = now` 后调 `storage.save(self)`；无 binding 则 no-op（backward-compat——让无持久化配置的 session 也能安全调用）。
- **`JsonFileStorage.save()`**：用 `os.open(path, O_WRONLY | O_CREAT | O_TRUNC, 0o600)` 以 user-only 权限写入——不经 Python 高层 `open()`，避免默认 644 权限下 session 数据被 group/other 读取。
- **`_path(session_id)`**：`(base_dir / f"{session_id}.json").resolve()`，然后 `is_relative_to(base_dir.resolve())` 校验——防止恶意 session_id 含 `../` 逃逸。
- **`cleanup(max_age_days=30, max_sessions=100)`**：先删超过 30 天未更新的，再删旧到只剩 100 个——LRU 式清理防止磁盘无限增长。

### QueryLoop 的 per-turn save

`core/loop.py` 在两个关键位置调 `_session.save_if_storage()`：

1. **end_turn 分支**（`:259`）：LLM 返回 end_turn，memory 抽取完成，assistant message 追加后保存。
2. **tool_use 分支**（`:339`）：工具执行完成，ages 更新 + auto_free_after 处理完成，保存。

效果：**任意轮 crash 后 resume，对话状态总是在某个完整 turn 的末尾**——不会出现 "tool_use 已调但 tool_result 未保存" 的半状态。

### Resume 流程

```python
# neoagent/agent.py:89-104
def resume(self, session_id: str, validate: bool = True) -> Session:
    if self._storage is None:
        raise RuntimeError("No SessionStorage configured")
    session = Session.resume(session_id, self._storage)
    if validate:
        self._validate_resume(session)
    return session
```

`_validate_resume` 做两类校验：

1. **workspace_path 存在性**：从 `session.metadata["workspace_path"]` 读，如果记录过但磁盘上不存在，发射 `SessionResumeWarningEvent(reason="workspace_missing", details=...)`。
2. **stale_session**：`updated_at` 距 now 超过 24h，发射 `reason="stale_session"` event。

**警告而非抛错**——让应用层自己决定"老 session 到底用不用"。

### Fork（分支会话）

```python
def fork(self, new_id: str | None = None) -> "Session":
    return Session(
        id=new_id or str(uuid.uuid4()),
        messages=copy.deepcopy(self.messages),
        state=copy.deepcopy(self.state),
        metadata=copy.deepcopy(self.metadata),
        created_at=datetime.now(timezone.utc).replace(tzinfo=None),
        updated_at=datetime.now(timezone.utc).replace(tzinfo=None),
    )
```

深拷贝全部数据（messages/state/metadata），生成新 id，**不 bind_storage**——fork 出的 session 默认不持久化，需要调用方显式 `bind_storage` 才加入 save_if_storage 循环。用途：A/B 测试不同 prompt、探索性分支继续、测试 "如果当时选择另一条路会怎样"。

### Storage Protocol 多实现

```python
@runtime_checkable
class SessionStorage(Protocol):
    def save(self, session: Session) -> None: ...
    def load(self, session_id: str) -> Session: ...
    def list_ids(self) -> list[str]: ...
    def delete(self, session_id: str) -> None: ...
```

`@runtime_checkable` 让 `isinstance(obj, SessionStorage)` 在运行时可用——用户可以轻松实现 `RedisSessionStorage`/`S3SessionStorage` 等自定义后端。`JsonFileStorage` 是内置默认，`NeoAgent.__init__` 检查 `storage` 参数和 `config.session_dir` 决定用哪个：显式 storage > session_dir 创建 JsonFileStorage > 全 None。

### 关键代码路径

- `neoagent/session.py:14-40` — FreedToolResult + SessionState 数据类
- `neoagent/session.py:42-95` — Session 数据类 + create/resume/save/fork
- `neoagent/session.py:76-80` — `save_if_storage` no-op fallback 的设计
- `neoagent/session.py:106-159` — JsonFileStorage 全部实现（含 0o600 + 路径逃逸 + cleanup）
- `neoagent/session.py:194-258` — `_session_to_dict` / `_session_from_dict` 序列化
- `neoagent/agent.py:89-126` — `resume()` + `_validate_resume`
- `neoagent/core/loop.py:259, 339` — per-turn save 调用点
- `neoagent/events.py:129-134` — `SessionResumeWarningEvent`

## 设计亮点

- **`save_if_storage` 设计避免"持久化 vs 无持久化"代码分叉**：`QueryLoop` 调 save_if_storage 永远安全，无 storage 时 no-op——循环体不需要 `if storage: save`，简化主路径。
- **Storage binding 与 Session 构造解耦**：Session 通过 `bind_storage(storage)` 后调用 `__post_init__` 写入 `_storage` 属性（非 dataclass 字段），避免 serialization 时污染 JSON——这是 dataclass 用作纯数据容器的最佳实践。
- **Fork 默认不持久化**：避免分叉时写回到 parent session 污染原数据；显式 bind_storage 的设计符合"明示副作用"原则。
- **0o600 + 路径逃逸双护栏**：session JSON 可能含敏感对话（用户身份、API tokens in prompts），0o600 保证 user-only；路径逃逸 (`is_relative_to` 校验) 防止构造 `session_id="../../etc/passwd"` 类攻击。
- **Warning 而非抛错的 resume 设计**：让应用层自由决定"workspace 没了要不要创建"/"session 太旧要不要用"，这是 CLI agent 和 web agent 对 stale 的不同容忍度要求决定的。
- **cleanup 双维度（天数 + 数量）**：先按 age、再按数量——保证即使用户每天用 100 次，磁盘总会稳定。
- **`SessionResumeWarningEvent.reason` 用 union 字符串**：`"workspace_missing"` / `"stale_session"`——预留后续加新原因的扩展点（如 `"provider_mismatch"`、`"schema_changed"`）。
- **full SessionState 持久化（含 freed_tool_results 原文）**：resume 后 LLM 可以 `recall_tool_result` 拿回 crash 前的原文，不需要重新执行——对 run_python 之类昂贵工具特别有价值。

## 局限性

- **`recalled_this_turn` 也被序列化但应该是 transient**：`_session_to_dict` 里没包含 recalled_this_turn——这点倒是做对了（它是每轮清空的 transient 状态），但 `_session_from_dict` 构造 SessionState 时直接用默认 `set()` 初始化。**Bug 风险**：如果在序列化检查器里改了实现，可能误加它。
- **JSON 文件可能很大**：freed_tool_results 存所有原文，长 run_python 输出 5KB × 100 次 = 500KB 的 JSON；无压缩、无分页。
- **无锁并发保护**：两个进程同时 `save(session)` 同一 id → 最后写赢，没有 file lock。multi-process 部署下可能数据损坏。
- **Workspace 校验过于简单**：只看 `workspace_path` 存在与否，不检查 cwd 文件是否与 session 记录时一致——用户在 workspace 里改了关键文件（被 agent 读过、content 存在 freed_tool_results 里），resume 后 agent 以为文件还是原貌。
- **session_dir cleanup 不在 QueryLoop 内自动调**：用户必须显式调 `JsonFileStorage(dir).cleanup()`，否则磁盘只增不删。
- **Fork 不复制 `_storage`**：这是故意的（防止污染父），但文档提示不足——用户看到 `fork()` 返回的 session 调 save_if_storage 没效果会困惑。
- **SessionStorage Protocol 没有 `exists(session_id)` 方法**：检查 session 是否存在必须 `try: load() except KeyError`，这是鸭子类型的惯例但接口不完整。
- **`_session_from_dict` 容错不足**：如果 JSON schema 变了（比如将来 SessionState 加新字段），旧 JSON resume 可能部分字段丢失但不报错——没有版本号字段，无法判断 schema 变更。
- **no `validate_resume` 钩子可扩展**：只有内置两种校验（workspace_missing + stale_session），用户想加"检查 api_key 是否仍有效"之类必须改 agent.py 源码或监听 event 后自行实现。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/session.py`, `neoagent/agent.py:89-126`, `neoagent/core/loop.py:259, 339`, `neoagent/events.py:129-134`
