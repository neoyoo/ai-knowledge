---
title: "runtime-state——agentscope"
category: L2
parent: "[[runtime-state]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "runtime-state"
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

agentscope 将运行时状态拆分为三个独立层：**进程级配置**（`_ConfigCls` + `ContextVar`）、**模块级状态树**（`StateModule` 递归序列化）、**会话持久化**（`SessionBase` 多后端适配）。三层正交设计使单进程内可运行多个隔离的 agent 实例，任意时刻都可将状态快照持久化，重启后无缝恢复。

---

## 架构分析

### 层一：进程级运行配置（`_ConfigCls`）

`src/agentscope/__init__.py` 在模块加载时创建唯一的全局单例 `_config`：

```python
_config = _ConfigCls(
    run_id=ContextVar("run_id", default=shortuuid.uuid()),
    project=ContextVar("project", default="UnnamedProject_At..."),
    name=ContextVar("name", default=...),
    created_at=ContextVar("created_at", default=...),
    trace_enabled=ContextVar("trace_enabled", default=False),
)
```

`_ConfigCls`（`src/agentscope/_run_config.py`）将每个字段包在 `ContextVar` 里，通过 property 读写。`ContextVar` 的语义保证了同一进程内不同 asyncio Task 可以各自持有独立的配置快照——这是 agentscope 支持多并发 agent 会话的底层机制。

`init()` 函数是唯一的公开修改入口，负责写入 project/name/run_id、初始化日志、向 Studio 注册运行实例、以及挂载 OpenTelemetry 追踪。

### 层二：模块级状态树（`StateModule`）

`src/agentscope/module/_state_module.py` 定义了可递归的状态容器基类：

```python
class StateModule:
    def __init__(self) -> None:
        self._module_dict = OrderedDict()    # 子 StateModule 注册表
        self._attribute_dict = OrderedDict() # 标量属性注册表

    def __setattr__(self, key: str, value: Any) -> None:
        if isinstance(value, StateModule):
            self._module_dict[key] = value   # 自动感知子模块
        super().__setattr__(key, value)

    def register_state(self, attr_name: str,
                       custom_to_json=None, custom_from_json=None) -> None:
        # 将标量属性纳入序列化范围
        self._attribute_dict[attr_name] = _JSONSerializeFunction(...)

    def state_dict(self) -> dict:
        # 递归序列化所有子模块 + 注册属性
        ...

    def load_state_dict(self, state_dict: dict, strict: bool = True) -> None:
        # 递归反序列化，strict=True 时键缺失抛异常
        ...
```

**继承链**如下：

```
StateModule
├── AgentBase          (src/agentscope/agent/_agent_base.py)
│   └── ReActAgent     (register_state("name"), register_state("_sys_prompt"))
├── MemoryBase         (src/agentscope/memory/_working_memory/_base.py)
│   └── InMemoryMemory (register_state("content"))  — 覆盖 state_dict/load_state_dict
└── LongTermMemoryBase (src/agentscope/memory/_long_term_memory/_long_term_memory_base.py)
```

`AgentBase` 本身继承 `StateModule`，其成员 `memory`（`MemoryBase` 子类）在赋值时自动被 `__setattr__` 钩住写入 `_module_dict`，因此 `agent.state_dict()` 会递归包含内存内容，无需手动处理嵌套。

### 层三：会话持久化（`SessionBase`）

`src/agentscope/session/_session_base.py` 定义抽象接口：

```python
class SessionBase:
    async def save_session_state(
        self, session_id: str, user_id: str = "",
        **state_modules_mapping: StateModule
    ) -> None: ...

    async def load_session_state(
        self, session_id: str, user_id: str = "",
        allow_not_exist: bool = True,
        **state_modules_mapping: StateModule
    ) -> None: ...
```

三个具体实现：

| 后端 | 文件 | 存储方式 | 特殊能力 |
|------|------|---------|---------|
| `JSONSession` | `_json_session.py` | 本地文件 `{user_id}_{session_id}.json` | `aiofiles` 异步 IO |
| `RedisSession` | `_redis_session.py` | Redis Hash key（带 prefix + TTL） | `GETEX` 原子刷新 TTL（sliding TTL） |
| `TablestoreSession` | `_tablestore_session.py` | Alibaba Tablestore metadata 字段 `__state__` | 懒加载初始化（asyncio.Lock 双检） |

所有后端统一调用模式：

```python
state_dicts = {
    name: state_module.state_dict()
    for name, state_module in state_modules_mapping.items()
}
# 序列化为 JSON 字符串后写入对应后端
```

### Msg 消息对象

`src/agentscope/message/_message_base.py` 定义了 `Msg`，作为 agent 间通信和状态中最核心的数据单位：

- 字段：`id`（shortuuid）、`name`、`role`（user/assistant/system）、`content`（`str | list[ContentBlock]`）、`metadata`、`timestamp`、`invocation_id`
- `content` 支持多模态块类型（`TextBlock`、`ToolUseBlock`、`ToolResultBlock`、`ImageBlock`、`AudioBlock`、`VideoBlock`）通过 TypedDict 定义
- 提供 `to_dict()` / `from_dict()` 双向序列化，供 `InMemoryMemory.state_dict()` 直接调用

---

## 关键代码路径

### 路径 1：状态自动感知注册

```
AgentBase.__init__()
  → super().__init__()                         # StateModule.__init__
  → self.memory = InMemoryMemory()
      └→ StateModule.__setattr__("memory", ...)  # 自动写入 _module_dict
  → self.register_state("name")               # 写入 _attribute_dict
```

### 路径 2：保存会话快照

```python
await session.save_session_state(
    session_id="user_1",
    agent1=agent1,   # 命名参数即 **state_modules_mapping
    agent2=agent2,
)
```

内部展开：

```
JSONSession.save_session_state(session_id, user_id, **mapping)
  → {name: module.state_dict() for name, module in mapping.items()}
      └→ AgentBase.state_dict()              # StateModule.state_dict()
          ├→ memory.state_dict()              # 子模块递归
          │   └→ InMemoryMemory.state_dict()
          │       → [[msg.to_dict(), marks], ...]
          └→ "name", "_sys_prompt"           # 注册的标量属性
  → json.dumps(state_dicts)
  → aiofiles.open(path, "w").write(json_str)
```

### 路径 3：恢复会话状态

```
JSONSession.load_session_state(session_id, user_id, **mapping)
  → aiofiles.open(path, "r").read() → json.loads()
  → for name, module in mapping.items():
      module.load_state_dict(states[name])
          └→ StateModule.load_state_dict()
              ├→ _module_dict[key].load_state_dict(sub_state)  # 递归子模块
              └→ setattr(self, key, from_json(value))          # 恢复标量
```

### 路径 4：Redis 滑动 TTL 刷新

```
RedisSession.load_session_state(...)
  → key = "{prefix}user_id:{uid}:session:{sid}:state"
  → if key_ttl is not None:
      data = await client.getex(key, ex=key_ttl)   # 原子 GET + 续期
    else:
      data = await client.get(key)
```

### 路径 5：进程级配置注入

```
agentscope.init(project="MyProject", run_id="abc")
  → _config.project = "MyProject"     # ContextVar.set()
  → _config.run_id = "abc"
  → if tracing_url: setup_tracing(endpoint)
                    _config.trace_enabled = True
```

---

## 设计亮点

### 1. `ContextVar` 驱动的并发安全全局配置

`_ConfigCls` 所有字段均包裹在 `ContextVar` 中，而非普通 class 属性。这意味着不同 asyncio Task（每个用户会话一个 Task）可以在不加锁的情况下各自维护独立的 `run_id`/`trace_enabled` 状态，天然支持多租户场景，无需线程锁或显式隔离。

### 2. `__setattr__` 钩子实现零侵入的子模块自动发现

`StateModule.__setattr__` 在每次赋值时检测 value 是否为 `StateModule` 子类，若是则自动写入 `_module_dict`。开发者为 agent 添加 memory 字段时无需任何额外注册调用，继承树的序列化深度自动向下传递。

### 3. 双维度序列化声明

通过 `register_state(attr_name)` 注册标量，通过 `__setattr__` 自动注册子模块，两条路径最终汇聚在 `state_dict()`。对于无法 JSON 直接序列化的对象（如 `Msg` 列表），允许传入 `custom_to_json` / `custom_from_json` 回调，做到通用性与扩展性兼顾。

### 4. 多后端统一接口 + 后端差异能力暴露

三后端共享 `SessionBase` 接口，调用方代码零感知后端差异。同时，Redis 后端额外暴露 `key_ttl`（sliding TTL 通过 GETEX 原子实现）和 `key_prefix`（多环境隔离），Tablestore 后端则通过 `_ensure_initialized()` 的双检锁（asyncio.Lock）延迟初始化，避免冷启动阻塞。

### 5. `allow_not_exist` 宽松加载语义

`load_session_state` 默认 `allow_not_exist=True`，不存在的 session 仅打日志不报错，使得"首次启动 / 状态恢复"路径统一，上层无需特殊处理冷启动场景。

---

## 局限性

### 1. 全量快照，无增量

每次 `save_session_state` 将所有注册模块的完整状态序列化一次写入，无 diff/patch 机制。对于长期对话（内存大量消息）或高频保存场景，JSON 序列化开销和存储体积会线性增长。

### 2. 进程级配置与多 Agent 隔离不彻底

`_config` 是模块级单例，虽然字段用了 `ContextVar`，但 `ContextVar` 的隔离边界是 asyncio Task，而非 agent 实例。同一 Task 内的多个 agent 会共享相同的 `run_id`/`project`，无法做到 agent 粒度的配置隔离。

### 3. `strict=True` 的脆弱性

`load_state_dict(strict=True)` 默认要求 state_dict 中存在模块注册的所有键。这在 agent 版本迭代（新增 `register_state` 字段）时会导致加载旧版本快照失败，缺乏前向兼容的 migration 机制。

### 4. 无生命周期事件

`StateModule` 和 `SessionBase` 均无 `on_save` / `on_load` / `on_init` 钩子，无法在状态变更时触发副作用（如通知、指标埋点），可观测性需额外在上层封装。

### 5. Tablestore 后端依赖外部托管服务

`TablestoreSession` 依赖阿里云 Tablestore 及 `tablestore-for-agent-memory` 专用 SDK，不可移植到其他关系型或文档型数据库，增加了部署耦合度。

---

## 来源

- `src/agentscope/_run_config.py` — 进程级配置类
- `src/agentscope/__init__.py` — `_config` 全局实例 + `init()` 函数
- `src/agentscope/module/_state_module.py` — `StateModule` 核心序列化逻辑
- `src/agentscope/session/_session_base.py` — 会话持久化抽象接口
- `src/agentscope/session/_json_session.py` — JSON 文件后端
- `src/agentscope/session/_redis_session.py` — Redis 后端（sliding TTL）
- `src/agentscope/session/_tablestore_session.py` — Tablestore 后端（懒加载）
- `src/agentscope/message/_message_base.py` — `Msg` 消息对象
- `src/agentscope/message/_message_block.py` — 多模态内容块类型
- `src/agentscope/memory/_working_memory/_base.py` — `MemoryBase`
- `src/agentscope/memory/_working_memory/_in_memory_memory.py` — `InMemoryMemory`
- `src/agentscope/agent/_agent_base.py` — `AgentBase`（继承 StateModule）
- `tests/session_test.py` — 端到端用法示例
- 源码版本：0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12
- 分析深度：源码级
