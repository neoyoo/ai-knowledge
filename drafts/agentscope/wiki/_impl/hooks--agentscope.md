---
title: "hooks——agentscope"
category: L2
parent: "[[hooks]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "hooks"
created: "2026-04-15"
confidence: high
---

## 概述

AgentScope 通过 Python 元类（metaclass）在编译期把 `reply`、`print`、`observe` 三个核心方法自动包裹进 pre/post 钩子链，支持实例级（instance-level）和类级（class-level）两个注册作用域，钩子函数同步异步均可，返回非 None 时即可修改传入参数或输出结果。

---

## 架构分析

### 整体设计

AgentScope 的 hooks 系统由三个文件协同构成：

| 文件 | 职责 |
|------|------|
| `types/_hook.py` | 定义 `AgentHookTypes` / `ReActAgentHookTypes` 类型别名 |
| `agent/_agent_meta.py` | 元类 `_AgentMeta` 在类创建时包裹目标方法；`_wrap_with_hooks` 实现 pre/post 链 |
| `agent/_agent_base.py` | `AgentBase` 声明所有 hook 容器（OrderedDict），提供注册/移除/清空 API |
| `hooks/_studio_hooks.py` | 内置 hook 实现：把消息转发给 AgentScope Studio |
| `hooks/__init__.py` | 公开 `_equip_as_studio_hooks` 便捷注册函数 |

### 六个 Hook 点（AgentBase）

```
pre_reply → reply() → post_reply
pre_print → print() → post_print
pre_observe → observe() → post_observe
```

ReActAgentBase 在此基础上额外增加：

```
pre_reasoning → _reasoning() → post_reasoning
pre_acting    → _acting()    → post_acting
```

### 两个作用域

- **实例级（instance-level）**：`_instance_{hook_type}_hooks`，`__init__` 中以 `OrderedDict()` 初始化，只影响当前 agent 实例
- **类级（class-level）**：`_class_{hook_type}_hooks`，类体中以 `OrderedDict()` 声明，影响该类的所有实例

注册顺序：实例级 hooks 先执行，类级 hooks 后执行。

### 元类自动包裹机制

`_AgentMeta.__new__` 在类创建时扫描 `attrs`，发现 `reply`、`print`、`observe` 中任一方法则替换为 `_wrap_with_hooks(original_func)` 的结果。这意味着子类只要定义这些方法，就自动获得 hook 能力，无需手动调用任何基类逻辑。

### 参数归一化

`_normalize_to_kwargs(func, self, *args, **kwargs)` 用 `inspect.signature` 把位置参数和关键字参数统一成一个 `dict`，传给 pre-hook，使 hook 函数无需关心调用方式的差异。

---

## 关键代码路径

### 路径一：元类包裹（类创建阶段）

```
_AgentMeta.__new__
  └─ for func_name in ["reply", "print", "observe"]:
       attrs[func_name] = _wrap_with_hooks(attrs[func_name])
           └─ async_wrapper(self, *args, **kwargs)
                ├─ _normalize_to_kwargs(original_func, self, *args, **kwargs)
                ├─ [pre-hook 链] for pre_hook in instance_hooks + class_hooks:
                │    modified = await _execute_async_or_sync_func(pre_hook, self, deepcopy(kwargs))
                │    if modified is not None: current_kwargs = modified
                ├─ current_output = await original_func(self, *args, **others, **kwargs)
                └─ [post-hook 链] for post_hook in instance_hooks + class_hooks:
                     modified = await _execute_async_or_sync_func(post_hook, self, deepcopy(kwargs), deepcopy(output))
                     if modified is not None: current_output = modified
```

关键文件：`agent/_agent_meta.py` L55-L144

### 路径二：注册 hook（运行时）

```python
# 实例级
agent.register_instance_hook(
    hook_type: AgentHookTypes,  # e.g. "pre_reply"
    hook_name: str,
    hook: Callable,
) -> None
    └─ hooks = getattr(self, f"_instance_{hook_type}_hooks")
       hooks[hook_name] = hook

# 类级
AgentBase.register_class_hook(
    hook_type: AgentHookTypes,
    hook_name: str,
    hook: Callable,
) -> None
    └─ hooks = getattr(cls, f"_class_{hook_type}_hooks")
       hooks[hook_name] = hook
```

关键文件：`agent/_agent_base.py` L533-L616

### 路径三：内置 Studio Hook 注册

```python
# hooks/__init__.py
def _equip_as_studio_hooks(studio_url: str) -> None:
    AgentBase.register_class_hook(
        "pre_print",
        "as_studio_forward_message_pre_print_hook",
        partial(
            as_studio_forward_message_pre_print_hook,
            studio_url=studio_url,
            run_id=_config.run_id,
        ),
    )
```

注册为类级 pre_print hook，所有 agent 实例打印时都会触发 HTTP 推送到 Studio。失败时有 3 次重试 + 优雅降级（`logger.warning`），不会因网络问题崩溃。

关键文件：`hooks/__init__.py` L17-L29，`hooks/_studio_hooks.py` L12-L58

### 路径四：同步/异步统一执行

```python
async def _execute_async_or_sync_func(func, *args, **kwargs):
    if await _is_async_func(func):
        return await func(*args, **kwargs)
    return func(*args, **kwargs)
```

关键文件：`_utils/_common.py` L133-L156

---

## 设计亮点

### 1. 元类透明注入，零侵入子类

子类只需 `class MyAgent(AgentBase)` 并实现 `reply`，`_AgentMeta` 就自动在编译期包裹 hooks。子类代码中完全看不到 hook 相关代码，维护成本为零。

### 2. 修改语义由返回值决定，而非强制

pre-hook 返回 `None` 表示"观察但不修改"，返回新 dict 才会替换传入参数；post-hook 同理。这使纯观测型 hook（如日志、追踪）无需任何返回语句，也不会意外影响主流程。

### 3. OrderedDict 保证执行顺序

hook 注册顺序即执行顺序，且 `name` 键实现覆盖语义——同名 hook 重新注册即替换，行为可预期。

### 4. 深拷贝隔离副作用

`_wrap_with_hooks` 传给 pre-hook 的是 `deepcopy(current_normalized_kwargs)`，传给 post-hook 的是 `deepcopy(kwargs)` 和 `deepcopy(output)`。hook 修改拷贝不会直接影响主流程对象；只有通过返回值显式声明才能传播修改。

### 5. 同步异步统一

`_execute_async_or_sync_func` 运行时检测函数类型，同步和异步 hook 可以混用。开发者不需要关心调用方是否在 event loop 中。

### 6. ReActAgent 在 reasoning/acting 粒度开放扩展点

`_ReActAgentMeta` 继承 `_AgentMeta`，额外在 `_reasoning` 和 `_acting` 方法上也注入 hooks，允许在推理步骤和工具调用步骤分别做干预（如注入 chain-of-thought 修正、记录 tool 调用日志）。

---

## 局限性

### 1. 类级 hook 存在继承共享问题（已知 bug，被注释掉的测试揭示）

`_class_pre_reply_hooks` 等 dict 以类属性形式声明在 `AgentBase` 上，子类 `ChildAgent` 注册 class hook 时实际写入的是父类共享的同一个 `OrderedDict`（除非子类显式重声明），导致不同子类的 class hook 相互污染。`hook_test.py` 中大量测试用例被注释掉（L619-L776），注释写明"The studio requires the hook inherited from AgentBase, we will solving this problem later"，说明团队清楚这个缺陷但尚未解决。

**实际影响**：目前 `_equip_as_studio_hooks` 注册在 `AgentBase` 本身，避开了继承问题；但用户若在子类上调用 `register_class_hook`，行为可能出乎意料。

### 2. 无拦截/短路语义（pre-hook 不能阻止主方法执行）

pre-hook 只能修改输入参数，无法返回一个值来"短路"主方法。若需要实现缓存、条件跳过等模式，需要通过其他机制（如在 pre-hook 中 raise 异常）绕行，不优雅。

### 3. hook 函数签名有隐式约定

pre-hook 必须接受 `(self, kwargs: dict)` 两个参数，post-hook 必须接受 `(self, kwargs: dict, output: Any)` 三个参数，且顺序固定。框架没有类型校验，签名不符时运行时报错，调试体验差。

### 4. 深拷贝开销

每个 hook 调用前都做 `deepcopy`，在大消息体（如长上下文）或高频调用场景下有性能代价，且框架层没有提供跳过拷贝的选项。

### 5. hooks 模块仅有单个内置实现

`hooks/` 目录目前只有 Studio 转发一个内置 hook，缺乏通用工具钩子（如消息日志、速率限制、token 计数等），开发者需自行实现并注册。

---

## 来源

- 源码版本：0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12
- 核心文件：
  - `src/agentscope/agent/_agent_meta.py`
  - `src/agentscope/agent/_agent_base.py`
  - `src/agentscope/agent/_react_agent_base.py`
  - `src/agentscope/hooks/__init__.py`
  - `src/agentscope/hooks/_studio_hooks.py`
  - `src/agentscope/types/_hook.py`
  - `tests/hook_test.py`
- 分析深度：源码级
