---
title: "hooks — neoagent"
category: L2
parent: "[[hooks]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: hooks
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的 hook 系统由 `HookManager`（`hooks.py` 263 行）实现**函数注册式 + 三态返回（allow/deny/modify）**的经典模型，4 种 hook type：`pre_tool_call`、`post_tool_call`、`pre_provider_call`、`post_provider_call`。独特点是**event 用 frozen dataclass + MappingProxyType 不可变包装**——handler 无法通过 in-place mutation 绕过 `HookResult.modify` 协议（必须显式返回 modify），同时 modify 通过 `dataclasses.replace` 构造新 event 传递给下游 handler（chain-of-responsibility 式）。另有正交的 `EventBus` 系统（`events.py`）用于**纯观测**（16 种事件类型，handler 不返回值、异常不中断）——两套完全独立：hooks 管拦截/修改，events 管广播/观测。

## 架构分析

### 两套系统的职责划分

| 维度 | HookManager（hooks.py） | EventBus（events.py） |
|------|------------------------|----------------------|
| 用途 | 拦截、修改、阻断 | 观测、审计、指标 |
| 返回值 | `HookResult(allow/deny/modify)` | `None`（handler 不返回） |
| 异常影响 | `log WARNING + skip handler`，chain 继续 | `log EXCEPTION`，其他 handler 继续 |
| 事件类型 | 4 种（pre/post × tool/provider） | 16 种（Provider/Tool/Compress/Memory/Skill/Turn/Worker/Task/SessionResume/ToolResult lifecycle） |
| 注入时机 | 同步嵌入 QueryLoop / ToolExecutor 关键路径 | 订阅-发布，感观测点无须修改主逻辑 |
| 执行顺序 | 按 priority 升序 + 注册序（`_HookEntry.seq` 稳定） | 按订阅顺序，list 迭代 |

### HookResult 三态协议

```python
@dataclass(frozen=True)
class HookResult:
    action: Literal["allow", "deny", "modify"]
    reason: str | None = None
    modified_data: dict | None = None

    @classmethod
    def allow(cls): ...
    @classmethod
    def deny(cls, reason: str = ""): ...
    @classmethod
    def modify(cls, data: dict, reason: str | None = None): ...
```

行为规则：

- **pre hook**：
  - `deny` → 短路返回 deny，后续 handler 不调用；主路径看到 deny 后拒绝执行（ToolExecutor 返回 error ToolResult / QueryLoop 返回 "[Hook denied: reason]" assistant message）
  - `modify` → 用 `dataclasses.replace(event, **{k: v for k,v in modified_data if k in fields})` 构造新 event，下一个 handler 看到新 event
  - `allow` → 继续
- **post hook**：
  - `deny` → **忽略**（代码注释："execution already happened"；下一 handler 看到未被修改的 event 继续）
  - `modify` → 同 pre
  - `allow` → 继续

pre chain 结束后，`HookManager.run_pre` 对比 original vs current event，用 `_extract_modified_data` 做 field-wise diff 生成最终 modify data（累积所有 modify 变更）。这是 chain-of-responsibility 的纯函数式实现——handler 不 mutate，只生成新 event。

### 4 种 Hook 事件类型

```python
@dataclass(frozen=True)
class PreToolCallEvent:
    tool_name: str
    tool_input: MappingProxyType  # __post_init__ 强制转为 immutable
    call_id: str

@dataclass(frozen=True)
class PostToolCallEvent:
    tool_name: str
    tool_input: MappingProxyType
    call_id: str
    result: str
    is_error: bool

@dataclass(frozen=True)
class PreProviderCallEvent:
    system: str
    messages: tuple            # __post_init__ 强制转为 tuple
    tools: tuple

@dataclass(frozen=True)
class PostProviderCallEvent:
    response: Response
    input_tokens: int
    output_tokens: int
```

**`__post_init__` 强制不可变**：即使 handler 作者错把 dict/list 传进去，dataclass 的 `object.__setattr__` 自动转为 `MappingProxyType(dict(...))` / `tuple(...)`——所以 `event.tool_input["key"] = "value"` 会抛 TypeError，强迫开发者走 `HookResult.modify({"tool_input": {...}})` 协议路径。

### Hook 在主路径的嵌入

**ToolExecutor._run_one**（`executor.py:67-150`）：

```python
# (1) permission check first
allowed = await self._permission.check(tool, validated_input)
if not allowed: return error

# (2) pre_tool_call hook after permission
if self._hook_manager:
    pre_event = PreToolCallEvent(tool_name, dict(call.input), call.id)
    pre_result = await self._hook_manager.run_pre("pre_tool_call", pre_event)
    if pre_result.action == "deny":
        return error("Hook denied")
    if pre_result.action == "modify" and pre_result.modified_data:
        new_input = pre_result.modified_data.get("tool_input", call.input)
        call = ToolCall(id=call.id, name=call.name, input=new_input)
        validated_input = tool.input_model.model_validate(call.input)
        # (3) RE-RUN permission check on modified input!
        allowed = await self._permission.check(tool, validated_input)
        if not allowed: return error
    
# (4) tool.execute(validated_input) ...

# (5) post_tool_call hook
if self._hook_manager:
    post_event = PostToolCallEvent(...)
    post_result = await self._hook_manager.run_post("post_tool_call", post_event)
    if post_result.action == "modify":
        new_output = post_result.modified_data.get("result", result.output)
        result = ToolResult(call_id=call.id, output=new_output, is_error=result.is_error)
```

**关键：pre_tool_call modify 后重新走权限检查**（executor.py:112-124）——注释明写 "hooks must not be able to bypass PermissionChecker by substituting a safe input with a dangerous one after the initial check passed"。典型防御场景：用户定义了 "只允许写 /tmp" 的 permission policy，hook 不能把 `{path: "/tmp/foo"}` 改成 `{path: "/etc/passwd"}`。

**QueryLoop pre_provider_call**（`loop.py:178-196`）：

```python
if self._hook_manager:
    pre_event = PreProviderCallEvent(system, list(msgs), list(schemas))
    pre_result = await self._hook_manager.run_pre("pre_provider_call", pre_event)
    if pre_result.action == "deny":
        # 直接生成 "[Hook denied: reason]" assistant message 退出循环
        turn = Turn(response=Message(role="assistant", content=[TextBlock(text=denial_msg)]), ...)
        return ConversationResult(turns=turns, reason="completed")
    if pre_result.action == "modify":
        _system = pre_result.modified_data.get("system", system)
        _msgs = pre_result.modified_data.get("messages", msgs)
        _schemas = pre_result.modified_data.get("tools", schemas)
```

### EventBus 发射点密度

16 种事件在代码里的发射位置：

| 事件 | 发射点 |
|------|--------|
| `ProviderRequestEvent` / `ProviderResponseEvent` | QueryLoop 每轮 provider 调用前后 |
| `ToolCallEvent` / `ToolResultEvent` | ToolExecutor._run_one |
| `CompressCheckEvent` / `CompressDoneEvent` / `CompressFallbackEvent` | QueryLoop 压缩分支 |
| `MemoryExtractEvent` | QueryLoop end_turn 分支 memory 触发点 |
| `SkillChangeEvent` | 定义了但未发射（TODO v3.2） |
| `TurnCompleteEvent` | QueryLoop 每轮结束（end_turn 和 tool_use 分支都发） |
| `ToolResultFreedEvent` | QueryLoop auto_free_after 触发点 |
| `ToolResultRecalledEvent` | 定义了但未发射 |
| `SessionResumeWarningEvent` | NeoAgent._validate_resume |
| `WorkerEvent` / `TaskDispatchEvent` / `TaskCompleteEvent` | 多 agent（multi/events.py、multi/tools/*） |

`EventBus.subscribe_all(handler)` 遍历 `_ALL_EVENT_TYPES` 列表订阅全部——用于 debug logging 或多 agent 的 `_setup_event_bubble`。

### 用户 API

```python
# 函数式注册
agent.hook("pre_tool_call", my_handler, priority=10)

# 装饰器
@agent.on("pre_tool_call")
async def my_handler(event):
    return HookResult.allow()

# EventBus 直接订阅
agent.event_bus.subscribe(ProviderRequestEvent, lambda e: print(e))
```

### 关键代码路径

- `neoagent/hooks.py:14-37` — `HookResult` 三态 + 工厂方法
- `neoagent/hooks.py:42-116` — 4 种事件 dataclass + `__post_init__` 不可变强制
- `neoagent/hooks.py:132-263` — `HookManager`（register/unregister/run_pre/run_post + `_apply_modify`/`_extract_modified_data`）
- `neoagent/tools/executor.py:93-124` — pre_tool_call + 二次权限检查
- `neoagent/core/loop.py:176-196` — pre_provider_call + deny 短路路径
- `neoagent/events.py:180-205` — EventBus subscribe/emit/subscribe_all
- `neoagent/agent.py:292-316` — `hook()`/`on()`/`unhook()` 三个用户 API

## 设计亮点

- **frozen dataclass + MappingProxyType/tuple 强制不可变**：handler 作者即使想"偷懒"用 `event.tool_input["key"] = val` 也会抛 TypeError——必须走 `HookResult.modify` 协议才能修改。这是从类型系统层面强制约定，不是靠文档规范。
- **pre hook modify 后二次权限检查**：防御 hook 被滥用为权限绕过通道。其他项目（如 Hermes Agent）的 hook 没见到这层防护，是 neoagent 的亮点。
- **`dataclasses.replace` 的 chain-of-responsibility**：每个 handler 收到的 event 是前面 handler modify 累积的结果，最终用 field-diff 生成合并 modify——handler 之间无共享状态，纯函数式组合。
- **Priority + seq 稳定排序**：`_HookEntry(priority, seq, handler)` 的 `seq` 来自 `self._seq` 单调递增计数器，`sort(key=lambda e: (e.priority, e.seq))` 保证同 priority 的 handler 按注册顺序执行——这是唯一"稳定排序"的实现（Python 3.7+ `list.sort` 本身稳定，但双 key 排序需要 tiebreaker）。
- **Hook vs Event 两套系统解耦**：hooks 承担"能否执行"决策，events 承担"发生了什么"广播；两者不互相依赖。这让添加新观测点无需修改 hook 代码，反之亦然。
- **Handler 异常不中断 chain**：`log.warning(exc_info=True)` + `continue`——一个坏 hook 不毒化整个 hook 流水线（生产安全）。
- **Post hook 的 deny 被文档化忽略**：代码注释明写 "ignored (execution already happened)"——开发者意图透明，不是"bug 但懒得修"。

## 局限性

- **事件粒度少**：Claude Code 有 28+ 事件类型、AgentScope 元类注入 `reply/print/observe/reasoning/acting` 5 个切入点，neoagent 只在 tool/provider 两个维度做 pre/post——想拦截 memory 抽取前后、压缩前后、skill 激活时必须改源码。
- **无 shell/LLM 评估型 hook**：Claude Code 的 shell 子进程 hook（让用户用 bash 写策略）和 OpenHarness 的 `prompt`/`agent` 类型（让 LLM 判断是否允许）都不支持；用户必须写 Python async function 才能 hook。
- **无 matcher 机制**：注册时无法声明 "只匹配工具名 `write_file` 的 pre_tool_call"——所有 pre_tool_call handler 都会被所有工具触发，handler 自己 filter。这对大量 hook 的场景是性能和可读性负担。
- **SkillChangeEvent 定义但未发射**：hooks.py 和 events.py 都声明了，但 PromptBuilder 没订阅 EventBus（TODO v3.2）——现阶段 skill 激活/停用无法被观测或拦截。
- **`ToolResultRecalledEvent` 同样未发射**：events.py 有定义，recall_tool_result.py 未引用——recall 行为无法被 observer 记录。
- **EventBus 不支持 async handler**：`emit` 里是 `handler(event)` 同步调用，如果 handler 是 coroutine 函数，会返回 coroutine 对象而不执行。multi agent 模块的 event bubble 被迫写成 sync handler。
- **`MappingProxyType` 的序列化开销**：每次 emit ToolCallEvent 都 `MappingProxyType(copy.deepcopy(call.input))`——高频工具调用场景的 deepcopy 开销不可忽略。
- **无 hook 优先级冲突检测**：两个不同 hook 同 priority 都做 modify 时，代码能跑但语义可能冲突（后写赢）——框架不警告。
- **pre_provider_call modify 的 schemas 不 revalidate**：executor 里 pre_tool_call 改 tool_input 后重验权限，但 loop 的 pre_provider_call 改 tools 后没验证 tool schemas 合法性——hook 可以偷偷往 tools 里加不存在的工具描述诱导 LLM 幻觉调用。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/hooks.py`, `neoagent/events.py`, `neoagent/tools/executor.py:93-150`, `neoagent/core/loop.py:176-212`, `neoagent/agent.py:292-316`
