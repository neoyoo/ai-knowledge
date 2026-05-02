---
title: "multi-agent——agentscope"
category: L2
parent: "[[multi-agent]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "multi-agent"
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

AgentScope 通过三个正交抽象实现多 agent 协作：`pipeline`（编排拓扑）、`MsgHub`（广播总线）、`A2AAgent`（跨进程互联）。本地协作靠 pipeline 驱动数据流，消息扇散靠 MsgHub 的订阅-广播机制，跨网络远程调用靠 Google A2A 协议标准化。`plan/` 模块提供单 agent 内的子任务跟踪，可与上述编排机制配合实现"计划-执行"模式。

---

## 架构分析

### 整体分层

```
用户代码
  ↓
Pipeline（编排层）     ←→  MsgHub（广播层）
  ↓                            ↓
AgentBase（每个 agent）
  ├── reply()                  ← 生成回复
  ├── observe()                ← 被动接收消息（不回复）
  └── _subscribers             ← MsgHub 注入的订阅者列表
  ↓
A2AAgent（远程 agent 代理）    ← 对端是另一进程的 A2A Server
```

### pipeline 模块

路径：`src/agentscope/pipeline/`

提供三种编排原语：

| 原语 | 类 / 函数 | 拓扑 |
|------|-----------|------|
| 顺序链 | `SequentialPipeline` / `sequential_pipeline()` | A → B → C，前一个输出作为下一个输入 |
| 扇出 | `FanoutPipeline` / `fanout_pipeline()` | 同一消息广播给 N 个 agent，收集各自输出 |
| 流式输出 | `stream_printing_messages()` | 异步生成器，聚合多个 agent 的中间打印消息 |
| 实时聊天室 | `ChatRoom` | 多个 `RealtimeAgent` 共享一条 asyncio.Queue，中央转发循环分发消息 |

`SequentialPipeline` 和 `FanoutPipeline` 都是可复用的类包装器，其核心逻辑分别委托给同模块的函数式版本。

### MsgHub 广播机制

路径：`src/agentscope/pipeline/_msghub.py`

MsgHub 是一个异步上下文管理器，进入时向所有参与者注入订阅关系，退出时清除。

关键机制：
- 每个 `AgentBase` 维护 `_subscribers: dict[str, list[AgentBase]]`，key 是 MsgHub 的 name（默认随机 uuid）
- `MsgHub.__aenter__` 调用每个 agent 的 `reset_subscribers(msghub_name, participants)`，将除自身外的所有成员写入订阅列表
- agent 的 `reply()` 完成后自动调用 `_broadcast_to_subscribers()`，将回复消息推送给所有订阅者的 `observe()` 方法
- `MsgHub.__aexit__` 调用每个 agent 的 `remove_subscribers(msghub_name)` 清除注册

这意味着：MsgHub 内任一 agent 的回复，会自动出现在其他所有参与者的历史上下文中，无需手动串联。

### A2A 跨进程协作

路径：`src/agentscope/agent/_a2a_agent.py` + `src/agentscope/a2a/`

`A2AAgent` 是一个本地代理对象，背后对接远程 A2A 协议服务器。它继承 `AgentBase`，对外表现与普通 agent 无差异，内部将消息翻译成 A2A 协议格式并通过 HTTP 发送。

**AgentCard 解析**（`src/agentscope/a2a/`）提供三种策略：
- `FileAgentCardResolver`：从本地 JSON 文件加载
- `WellKnownAgentCardResolver`：从 `/.well-known/` URL 发现
- `NacosAgentCardResolver`：从 Nacos 服务注册中心动态拉取，支持版本锁定

### PlanNotebook 子任务管理

路径：`src/agentscope/plan/`

`PlanNotebook` 是一个 `StateModule`，向单个 agent 暴露工具函数集，让 agent 通过 tool call 管理自己的执行计划。它与上层编排无直接耦合，可以插入任意 agent。

核心数据模型：
- `Plan`：包含 id、name、description、expected_outcome、subtasks 列表、state 状态机（todo/in_progress/done/abandoned）
- `SubTask`：细粒度任务单元，同样有四态状态机，记录实际 outcome

状态机约束：
- 同一时刻只能有一个 subtask 处于 `in_progress`
- 切换某 subtask 到 `in_progress` 之前，其前序 subtask 必须全部 done 或 abandoned
- `finish_subtask()` 完成后自动激活下一个 subtask

---

## 关键代码路径

### 1. MsgHub 订阅-广播链路

```python
# 进入 MsgHub 时注入订阅
async def __aenter__(self) -> "MsgHub":
    # _msghub.py:73
    self._reset_subscriber()
    if self.announcement is not None:
        await self.broadcast(msg=self.announcement)
    return self

def _reset_subscriber(self) -> None:
    # _msghub.py:89
    if self.enable_auto_broadcast:
        for agent in self.participants:
            agent.reset_subscribers(self.name, self.participants)

# AgentBase 收到注册时，将除自身外的成员写入订阅列表
def reset_subscribers(self, msghub_name: str, subscribers: list["AgentBase"]) -> None:
    # _agent_base.py:715
    self._subscribers[msghub_name] = [_ for _ in subscribers if _ != self]

# agent 回复后自动广播
async def _broadcast_to_subscribers(self, msg: Msg | list[Msg] | None) -> None:
    # _agent_base.py:469
    for subscribers in self._subscribers.values():
        for subscriber in subscribers:
            await subscriber.observe(broadcast_msg)

# 退出 MsgHub 时清除订阅
async def __aexit__(self, *args, **kwargs) -> None:
    # _msghub.py:83
    if self.enable_auto_broadcast:
        for agent in self.participants:
            agent.remove_subscribers(self.name)
```

### 2. fanout_pipeline 并发执行

```python
async def fanout_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
    enable_gather: bool = True,
    **kwargs: Any,
) -> list[Msg]:
    # _functional.py:47
    if enable_gather:
        tasks = [
            asyncio.create_task(agent(deepcopy(msg), **kwargs))
            for agent in agents
        ]
        return await asyncio.gather(*tasks)
    else:
        return [await agent(deepcopy(msg), **kwargs) for agent in agents]
```

每个 agent 收到的是消息的深拷贝，避免并发修改共享状态。

### 3. sequential_pipeline 链式传递

```python
async def sequential_pipeline(
    agents: list[AgentBase],
    msg: Msg | list[Msg] | None = None,
) -> Msg | list[Msg] | None:
    # _functional.py:10
    for agent in agents:
        msg = await agent(msg)
    return msg
```

### 4. A2AAgent.reply() 远程调用链

```python
async def reply(self, msg: Msg | list[Msg] | None = None, **kwargs) -> Msg:
    # _a2a_agent.py:177

    # 1. 合并本地观察到的历史消息与当前输入
    msgs_list = self._observed_msgs
    if msg is not None:
        msgs_list.append(msg) / msgs_list.extend(msg)

    # 2. 创建 A2A 客户端
    client = self._a2a_client_factory.create(card=self.agent_card)

    # 3. 将 AgentScope Msg 转换为 A2A Message 格式
    a2a_message = await self.formatter.format([_ for _ in msgs_list if _])

    # 4. 流式发送，处理 A2AMessage 和 Task 两种响应类型
    async for item in client.send_message(a2a_message):
        if isinstance(item, A2AMessage):
            response_msg = await self.formatter.format_a2a_message(...)
        elif isinstance(item, tuple):  # Task
            for _ in await self.formatter.format_a2a_task(...):
                response_msg = _

    # 5. 清空观察历史
    self._observed_msgs.clear()
    return response_msg
```

### 5. PlanNotebook 子任务状态流转

```python
async def finish_subtask(self, subtask_idx: int, subtask_outcome: str) -> ToolResponse:
    # _plan_notebook.py:548
    # 验证前序任务全部完成
    for idx, subtask in enumerate(self.current_plan.subtasks[0:subtask_idx]):
        if subtask.state not in ["done", "abandoned"]:
            return ToolResponse(...)  # 返回错误

    # 标记完成
    self.current_plan.subtasks[subtask_idx].finish(subtask_outcome)

    # 自动激活下一个子任务
    if subtask_idx + 1 < len(self.current_plan.subtasks):
        self.current_plan.subtasks[subtask_idx + 1].state = "in_progress"

    await self._trigger_plan_change_hooks()
```

---

## 设计亮点

### 1. 上下文管理器自动管理订阅生命周期

MsgHub 用 `async with` 包裹对话轮次，进入时注入订阅，退出时清除。这避免了"忘记清理订阅导致消息泄露到后续轮次"的常见陷阱。在多轮对话中，不同轮次可以有不同的参与者组合，只需嵌套/重新进入 MsgHub 即可。

### 2. 深拷贝保证并发安全

`fanout_pipeline` 在派发给每个 agent 前执行 `deepcopy(msg)`，确保并发执行的 agent 不会因共享同一消息对象而出现竞争条件。这是一个简单但关键的设计选择，许多框架忽略了这一点。

### 3. A2A 标准化使远程 agent 对本地透明

`A2AAgent` 完全继承 `AgentBase` 的接口，远程 agent 和本地 agent 对编排层无差异。`pipeline`、`MsgHub` 等机制无需为远程场景做特殊处理。服务发现通过 AgentCard Resolver 三策略（文件/Well-Known/Nacos）解耦，本地开发用文件，生产用 Nacos。

### 4. PlanNotebook 的 hook 机制用于 UI 同步

`PlanNotebook` 在每次计划状态变更后触发 `_plan_change_hooks`，允许外部注册回调（如将计划状态推送到前端 UI）。这将"状态持久化/展示"与"计划执行逻辑"彻底解耦。

### 5. DefaultPlanToHint 用状态机生成差异化 Prompt

`DefaultPlanToHint.__call__()` 根据当前计划状态（全 todo / 有 in_progress / 有 done 但无 in_progress / 全完成）生成不同的 `<system-hint>` 提示，引导 agent 执行不同的下一步动作。这是一种将执行状态机外置为 Prompt 工程的做法，相比在 agent 系统提示中硬编码流程更灵活。

### 6. `stream_printing_messages` 统一聚合多 agent 流式输出

通过共享 `asyncio.Queue`，将多个并发 agent 的中间打印消息统一汇聚成一个异步生成器，上层可以像处理单个 agent 输出一样处理多 agent 的混合流。

---

## 局限性

### 1. A2AAgent 不支持多 agent 上下文传递

源码注释明确说明：A2A 协议仅支持双角色（user/assistant）会话。AgentScope 的 `name` 字段在 A2A 协议中无对应，需要服务端自行处理。跨 agent 的对话历史无法原生通过 A2A 协议传递给远端。

### 2. PlanNotebook 的计划是串行执行的

`update_subtask_state` 强制要求前序 subtask 全部完成才能将某个 subtask 切换到 `in_progress`，且同一时刻只能有一个 in_progress subtask。这意味着 PlanNotebook 不支持并行子任务，与 pipeline 层的并发能力形成断层。

### 3. MsgHub 广播粒度是"全员"

所有参与者都会收到其他人的每条消息，无法做到选择性路由（如 A 只发给 B，不发给 C）。如果需要非全连接的消息拓扑，需要退出 MsgHub 手动调用 `observe()`。

### 4. ChatRoom 只适用于 RealtimeAgent

`ChatRoom` 与 `RealtimeAgent` 强绑定（基于 WebSocket 实时 API 协议），普通 LLM agent 无法加入 ChatRoom。

### 5. InMemoryPlanStorage 无持久化

默认的计划存储是内存字典（`OrderedDict`），进程重启后计划历史丢失。虽有 `PlanStorageBase` 抽象供扩展，但官方未提供数据库实现。

### 6. plan 状态刷新逻辑不完整

`Plan.refresh_plan_state()` 的注释明确标注 `# TODO: Handle the plan state much more formally.`，当前实现只在 todo↔in_progress 之间切换，不会自动将计划标记为 done（需要 agent 显式调用 `finish_plan`）。

---

## 来源

- 源码版本：0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12
- 分析深度：源码级
- 核心文件：
  - `src/agentscope/pipeline/_msghub.py`
  - `src/agentscope/pipeline/_functional.py`
  - `src/agentscope/pipeline/_class.py`
  - `src/agentscope/pipeline/_chat_room.py`
  - `src/agentscope/agent/_a2a_agent.py`
  - `src/agentscope/a2a/_base.py` / `_file_resolver.py` / `_well_known_resolver.py` / `_nacos_resolver.py`
  - `src/agentscope/plan/_plan_model.py`
  - `src/agentscope/plan/_plan_notebook.py`
  - `src/agentscope/plan/_storage_base.py` / `_in_memory_storage.py`
  - `examples/workflows/multiagent_debate/main.py`
  - `examples/workflows/multiagent_concurrent/main.py`
  - `examples/agent/a2a_agent/main.py`
