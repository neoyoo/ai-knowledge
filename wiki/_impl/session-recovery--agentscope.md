---
title: "session-recovery——agentscope"
category: L2
parent: "[[session-recovery]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "session-recovery"
created: "2026-04-15"
confidence: high
---

## 概述

AgentScope 的 session-recovery 以「状态模块树序列化 + 可插拔存储后端」为核心：所有需要持久化的 agent 状态（内存、系统提示、计划、工具激活状态等）统一继承 `StateModule`，通过 `state_dict()` / `load_state_dict()` 协议序列化；`SessionBase` 定义存储接口，目前提供三种后端（JSON 文件、Redis、Alibaba Tablestore），调用方选择后端后只需一次 `save_session_state` / `load_session_state` 即可将任意数量 agent 的完整状态原子地保存或恢复。

---

## 架构分析

### 核心抽象：StateModule 树

`StateModule`（`agentscope/module/_state_module.py`）是整个序列化体系的根基。它是一个可继承的 mixin，重载了 `__setattr__` 以自动追踪所有**子 StateModule 属性**（存入 `_module_dict`），并通过 `register_state()` 显式注册**基础类型属性**（存入 `_attribute_dict`）。

```
StateModule
├── _module_dict: {attr_name → StateModule 子对象}   # 子模块，递归序列化
└── _attribute_dict: {attr_name → _JSONSerializeFunction}  # 基础属性，支持自定义序列化函数
```

`state_dict()` 递归遍历 `_module_dict` 和 `_attribute_dict`，生成可 JSON 序列化的嵌套字典。`load_state_dict()` 对称地反序列化，strict 模式下缺键报错，非 strict 模式下跳过。

**继承关系**（生产代码中的实际继承链）：

```
StateModule
├── AgentBase          (agentscope/agent/_agent_base.py)
│   └── ReActAgent     (agentscope/agent/_react_agent.py)
│       ├── register_state("name")
│       └── register_state("_sys_prompt")
├── MemoryBase         (agentscope/memory/_working_memory/_base.py)
│   └── InMemoryMemory
│       ├── register_state("_compressed_summary")  # 来自 MemoryBase
│       └── register_state("content")              # 来自 InMemoryMemory
├── Toolkit            (agentscope/tool/_toolkit.py)
│   └── 自定义 state_dict/load_state_dict：仅序列化 active_groups
└── PlanNotebook       (agentscope/plan/_plan_notebook.py)
    └── register_state("current_plan", custom_to_json=..., custom_from_json=...)
```

由于 `AgentBase` 继承 `StateModule`，且 `InMemoryMemory`、`Toolkit`、`PlanNotebook` 也是 `StateModule`，当它们作为 agent 的属性被赋值时，会自动注册进 agent 的 `_module_dict`，`state_dict()` 调用自动递归。

### 存储层：SessionBase 接口

`SessionBase`（`agentscope/session/_session_base.py`）定义了两个抽象异步方法：

```python
class SessionBase:
    @abstractmethod
    async def save_session_state(
        self, session_id: str, user_id: str = "", **state_modules_mapping: StateModule
    ) -> None: ...

    @abstractmethod
    async def load_session_state(
        self, session_id: str, user_id: str = "",
        allow_not_exist: bool = True, **state_modules_mapping: StateModule
    ) -> None: ...
```

`**state_modules_mapping` 的设计允许**多个 agent 同时传入**，每个 agent 以关键字参数名作为键，批量保存或恢复，示例：

```python
await session.save_session_state(
    session_id="user_1",
    agent1=agent1,
    agent2=agent2,
)
```

### 三种存储后端

| 后端 | 键/路径规则 | TTL 支持 | 依赖 |
|------|------------|---------|------|
| `JSONSession` | `{save_dir}/{user_id}_{session_id}.json` | 无 | `aiofiles` |
| `RedisSession` | `{prefix}user_id:{uid}:session:{sid}:state` | 滑动 TTL（`key_ttl` 参数） | `redis[async]` |
| `TablestoreSession` | Alibaba Tablestore 表 `agentscope_session` 的 `metadata.__state__` 字段 | Tablestore 自身管理 | `tablestore`, `tablestore-for-agent-memory` |

`TablestoreSession` 额外引入了懒加载初始化模式（`_ensure_initialized()`），通过 `asyncio.Lock` 防止并发初始化竞争。

### 数据流向

```
调用方
  ↓  save_session_state(session_id, user_id, agent1=..., agent2=...)
SessionBase 实现
  ↓  对每个 StateModule 调用 state_module.state_dict()
  ↓  → {"agent1": {...}, "agent2": {...}}
  ↓  json.dumps(state_dicts)
存储后端 (文件 / Redis key / Tablestore metadata.__state__)

调用方
  ↓  load_session_state(session_id, user_id, allow_not_exist=True, agent1=..., agent2=...)
存储后端
  ↓  读取 JSON 字符串
  ↓  json.loads → {"agent1": {...}, "agent2": {...}}
  ↓  对每个 StateModule 调用 state_module.load_state_dict(states[name])
StateModule 树就地恢复（in-place mutation）
```

---

## 关键代码路径

### 路径 1：JSON 文件保存

```
JSONSession.save_session_state(session_id, user_id, **state_modules_mapping)
  → _get_save_path(session_id, user_id)
      → os.makedirs(save_dir, exist_ok=True)
      → return f"{save_dir}/{user_id}_{session_id}.json"
  → {name: module.state_dict() for name, module in state_modules_mapping.items()}
      → StateModule.state_dict()
          → 递归 _module_dict 中各子 StateModule.state_dict()
          → 遍历 _attribute_dict，应用 custom_to_json（若有）
  → aiofiles.open(path, "w").write(json.dumps(state_dicts))
```

关键文件：`agentscope/session/_json_session.py:47-79`

### 路径 2：Redis 滑动 TTL 加载

```
RedisSession.load_session_state(session_id, user_id, allow_not_exist, **mapping)
  → _get_session_key(session_id, user_id)
      → key_prefix + "user_id:{uid}:session:{sid}:state"
  → 若 key_ttl is not None:
        data = await client.getex(key, ex=key_ttl)   # 原子读 + 刷新 TTL
    否则:
        data = await client.get(key)
  → json.loads(data) → states
  → for name, module in mapping.items():
        if name in states:
            module.load_state_dict(states[name])    # in-place 恢复
```

关键文件：`agentscope/session/_redis_session.py:129-178`

### 路径 3：Tablestore 懒加载初始化

```
TablestoreSession.save_session_state(session_id, user_id, **mapping)
  → _ensure_initialized()
      → asyncio.Lock 保护
      → AsyncMemoryStore(...) 实例化
      → await memory_store.init_table()
      → await memory_store.init_search_index()
      → self._initialized = True
  → state_dicts = {name: module.state_dict() ...}
  → serialized_state = json.dumps(state_dicts)
  → TablestoreSessionModel(session_id, user_id, metadata={"__state__": serialized_state})
  → await memory_store.update_session(tablestore_session)
```

关键文件：`agentscope/session/_tablestore_session.py:91-169`

### 路径 4：InMemoryMemory 自定义序列化

`InMemoryMemory` 重写了 `state_dict()` 和 `load_state_dict()`，将 `content: list[tuple[Msg, list[str]]]` 转为 `[[msg.to_dict(), marks], ...]`：

```python
# _in_memory_memory.py:273-278
def state_dict(self) -> dict:
    return {
        **super().state_dict(),
        "content": [[msg.to_dict(), marks] for msg, marks in self.content],
    }

# 反序列化时兼容旧格式（list[dict] → tuple[Msg, list]）
def load_state_dict(self, state_dict: dict, strict: bool = True) -> None:
    for item in state_dict.get("content", []):
        if isinstance(item, (tuple, list)) and len(item) == 2:
            msg = Msg.from_dict(item[0])
            self.content.append((msg, item[1]))
        elif isinstance(item, dict):  # 向下兼容旧版本
            self.content.append((Msg.from_dict(item), []))
```

关键文件：`agentscope/memory/_working_memory/_in_memory_memory.py:273-305`

### 路径 5：PlanNotebook 的 Pydantic 自定义序列化

```python
# _plan_notebook.py:226-230
self.register_state(
    "current_plan",
    custom_to_json=lambda _: _.model_dump() if _ else None,
    custom_from_json=lambda _: Plan.model_validate(_) if _ else None,
)
```

`Plan` 是 Pydantic 模型，`model_dump()` / `model_validate()` 作为自定义序列化函数注入 `_attribute_dict`，实现复杂对象的无损序列化。

---

## 设计亮点

### 1. PyTorch-style StateModule 协议
`state_dict()` / `load_state_dict()` API 直接借鉴 PyTorch 的模型参数序列化范式，对有 ML 背景的开发者零学习成本。树状递归序列化配合 `__setattr__` 自动追踪子模块，无需手写序列化逻辑，仅需 `register_state("attr_name")` 声明哪些基础属性需要持久化。

### 2. 自定义序列化函数解耦复杂类型
`register_state(attr, custom_to_json, custom_from_json)` 允许任意不可 JSON 序列化的对象（Pydantic 模型、dataclass、enum 等）通过注入 lambda 实现无侵入序列化。`PlanNotebook.current_plan` 是典型案例，Pydantic 的 `model_dump` / `model_validate` 直接作为钩子函数。

### 3. Redis GETEX 实现滑动 TTL
`RedisSession.load_session_state` 在 TTL 模式下使用 `GETEX`（GET + EXPIRE 原子操作），每次读取自动刷新过期时间，实现活跃用户 session 不过期、不活跃用户 session 自动清理的滑动窗口行为，一行代码替代了 GET + EXPIRE 两步操作及其竞态。

### 4. Tablestore 双重懒加载 + asyncio.Lock
Tablestore 的初始化涉及网络 IO（建表、建索引），`TablestoreSession` 将初始化推迟到第一次真正使用时，且用 `asyncio.Lock` + double-check 防止并发初始化：
```python
async def _ensure_initialized(self):
    if self._initialized:
        return
    async with self._init_lock:
        if self._initialized:   # double-check
            return
        ...
```
这是异步环境下经典的懒加载安全模式。

### 5. allow_not_exist 优雅冷启动
三个后端的 `load_session_state` 均支持 `allow_not_exist=True`（默认值），新 session 首次加载时静默跳过而不抛异常，天然支持「load → use → save」的冷启动-热启动统一路径，调用方无需区分首次与恢复场景。

### 6. 多 agent 批量操作
`**state_modules_mapping` 关键字参数设计允许单次调用同时保存/恢复多个 agent 的状态，避免多次 IO，尤其对 Redis 和 Tablestore 网络后端意义显著。

---

## 局限性

### 1. 无事务性：多 agent 保存非原子
`save_session_state` 内部对多个 `state_module.state_dict()` 的序列化是纯内存操作，最终写入是单次 `json.dumps` + 一次 IO，因此多 agent 状态在**单次调用内**是原子的。但如果调用方多次调用（如先保存 agent1 再保存 agent2 到不同 session key），没有跨调用的事务保证。此外，`TablestoreSession.save_session_state` 使用 `update_session`，若进程在写入途中崩溃，状态可能部分丢失。

### 2. 全量覆盖式持久化，无增量 diff
每次 `save_session_state` 都是全量序列化 + 全量写入，不支持只保存变更部分。对于 memory 内容大（数百条消息）或 plan 复杂的 agent，存储开销随 session 增长线性增加。

### 3. 严格依赖 register_state 的手动声明
开发者必须在构造函数中**手动调用** `register_state("attr_name")` 才能让属性参与序列化；遗漏声明的属性不会有任何警告，只是默默不被持久化。相比 `__slots__` 或注解驱动的自动扫描，这更容易导致遗漏。

### 4. no checkpoint：粒度为整个 session
框架只提供手动触发的 `save/load`，没有内置的 checkpoint 机制（如每 N 轮自动保存、reply 后自动落盘）。对话过程中崩溃会丢失上次 save 之后的所有状态，依赖调用方在合适时机主动调用。

### 5. Tablestore 后端是阿里云独家
`TablestoreSession` 深耦合 `tablestore` 和 `tablestore_for_agent_memory` 两个阿里云专属 SDK，非阿里云用户无法使用，也不提供通用的关系型数据库（如 PostgreSQL）后端。

### 6. strict=True 下的版本兼容性脆弱
`load_state_dict(state_dict, strict=True)`（默认）在 state_dict 中缺少任何已注册 key 时抛 `KeyError`，版本迭代时如果新增了 `register_state` 字段，旧版本存储的 state 将无法加载（除非降为 `strict=False`）。`InMemoryMemory` 通过手动向下兼容旧格式（`isinstance(item, dict)` 分支）处理此问题，但框架层无统一的版本迁移机制。

---

## 来源

- 源码版本：`0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12`
- 分析深度：源码级
- 关键文件：
  - `agentscope/module/_state_module.py`
  - `agentscope/session/_session_base.py`
  - `agentscope/session/_json_session.py`
  - `agentscope/session/_redis_session.py`
  - `agentscope/session/_tablestore_session.py`
  - `agentscope/memory/_working_memory/_base.py`
  - `agentscope/memory/_working_memory/_in_memory_memory.py`
  - `agentscope/agent/_agent_base.py`
  - `agentscope/agent/_react_agent.py`
  - `agentscope/plan/_plan_notebook.py`
  - `agentscope/tool/_toolkit.py`
  - `tests/session_test.py`
