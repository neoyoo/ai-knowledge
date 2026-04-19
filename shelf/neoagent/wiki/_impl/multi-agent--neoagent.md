---
title: "multi-agent — neoagent"
category: L2
parent: "[[multi-agent]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: multi-agent
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的 multi-agent 模块实现了经典的 **Orchestrator + Worker 层级结构**（`multi/` 6 文件 844 行）：`Orchestrator` 是 brain = 一个 NeoAgent 实例，通过 5 个内置工具（spawn_worker/delegate_task/cancel_task/list_workers/list_tasks）让 LLM 自己决定如何拆解任务；`WorkerCard` 是 frozen dataclass 描述静态 worker（从 `.md` 文件 YAML frontmatter 解析，包含 name/description/instruction/tags/model/tools）；`Task/TaskResult/TaskTracker` 三元组管理任务生命周期（asyncio.timeout + CancelledError + 工作摘要提取）。独特点：**event bubble 机制**（`_setup_event_bubble` 订阅 worker event bus 全部事件，包成 `WorkerEvent(worker_name, task_id, depth, inner)` 再发射到 orchestrator）、**递归深度限制 + `SpawnWorkerTool(depth+1)`** 让子 worker 也能 spawn 子子 worker 直到 max_depth、**semaphore 并发上限** 控制整体资源消耗。

## 架构分析

### 核心组件五元组

| 组件 | 职责 | 代码位置 |
|------|------|---------|
| `Orchestrator` | 协调器 + 5 个内部工具注册 + brain NeoAgent | `orchestrator.py` (118 行) |
| `WorkerPool` | 静态 WorkerCard 注册表 (thread-safe) | `worker.py:27-66` |
| `WorkerCard` | 不可变 worker 描述（frozen dataclass） | `worker.py:15-24` |
| `Task/TaskResult/TaskTracker` | 任务生命周期 | `task.py` (424 行) |
| `SpawnWorker/DelegateTask/CancelTask/ListWorkers/ListTasks` | 5 个内置工具 | `multi/tools/*.py` |
| `_setup_event_bubble` | worker event → orchestrator event 适配 | `events.py` (63 行) |

### 启动流程

```python
orch = Orchestrator(NeoAgentConfig(api_key="..."))
# 内部：
#   self._brain = NeoAgent(config)
#   self._worker_pool = WorkerPool()
#   self._task_tracker = TaskTracker()
#   self._tool_pool: dict[str, BaseTool] = {}
#   self._semaphore = asyncio.Semaphore(max_concurrent_workers=5)
#   self._event_bus = self._brain._event_bus       # 共享 brain 的 bus
#   _register_internal_tools() → 给 brain 注册 5 个工具
#
# 5 tools: SpawnWorkerTool(orch, depth=0), DelegateTaskTool(orch, depth=0),
#          CancelTaskTool(orch), ListWorkersTool(orch), ListTasksTool(orch)

orch.register_worker(WorkerCard(name="reviewer", ...))
orch.load_workers("./workers/")  # 目录下所有 *.md 用 parse_worker_md 解析
orch.register_tool(ReadTool())   # 加入 _tool_pool 供 worker 按 card.tools 白名单领取

result = await orch.run("Review these three files for security issues")
# = await self._brain.chat(message)
# brain LLM 看到 5 个工具，决定用哪个拆解
```

### WorkerCard 的 .md 格式

```markdown
---
name: code-reviewer
description: 代码安全审查专家
model: sonnet                 # None = 继承 orchestrator config
tags: [security, review]
tools: [Read, Grep, Glob]     # 白名单，必须在 orchestrator._tool_pool 中
---

你是一个资深的代码安全审查员，专注于...
```

`parser.py:parse_worker_md` 用 `content.split("---", maxsplit=2)` 拆 frontmatter + body；`yaml.safe_load` 解析 YAML；body 作为 `instruction`（system prompt）。校验 `name` 必填，`tags/tools` 默认空 tuple。

### spawn_worker vs delegate_task 的差异

| 维度 | `spawn_worker` | `delegate_task` |
|------|----------------|-----------------|
| worker 来源 | 动态解析 `md_definition` 字符串，临时生成 WorkerCard | 查 `WorkerPool.find(worker_name)` 取静态 card |
| 生命周期 | 一次性（不加入 WorkerPool） | 复用 |
| 使用场景 | LLM 临时发明角色（"创建一个叫 'security-auditor' 的 agent"） | LLM 从已注册 worker 里选一个 |
| 代码 | `spawn_worker.py:56-128` | `delegate_task.py:63-148` |

两者共用底层流程：`_create_worker_agent(card, orchestrator, depth+1, task_id)` → `_run_worker(agent, task, tracker)` → `TaskResult` → `_format_task_result`。差异只在"card 从哪来"。

### _create_worker_agent 的构造

```python
def _create_worker_agent(card, orchestrator, depth, task_id="") -> NeoAgent:
    orch_config = orchestrator.config
    worker_model = card.model if card.model is not None else orch_config.model
    
    # 继承 orchestrator 的 api_key/provider/base_url/budgets
    worker_config = NeoAgentConfig(
        api_key=orch_config.api_key,
        model=worker_model,
        provider=orch_config.provider,
        ...
        system_prompt=card.instruction,   # worker 的 system prompt 来自 card.instruction
    )
    agent = NeoAgent(worker_config)
    
    # 按 card.tools 白名单领取 tool_pool 里的工具（copy.copy 浅拷贝实例）
    for tool_name in card.tools:
        tool = tool_pool.get(tool_name)
        if tool is None:
            logger.debug("tool %r not found for worker %r — skipping", tool_name, card.name)
            continue
        agent.register_tool(copy.copy(tool))
    
    # 只要 depth < max_depth，子 worker 也能 spawn（递归深度限制）
    if depth < orchestrator.max_depth:
        agent.register_tool(SpawnWorkerTool(orchestrator=orchestrator, depth=depth))
    
    # 事件冒泡
    _teardown = _setup_event_bubble(agent, orchestrator, card.name, task_id, depth)
    agent._bubble_teardown = _teardown  # 存 teardown 给外层调用者
    
    return agent
```

**关键**：worker 不包含 `DelegateTaskTool` / `CancelTaskTool` / `ListWorkersTool` / `ListTasksTool`——只有 `SpawnWorkerTool`，并且 depth 累加后判断是否 < max_depth 才注册。这意味着 **只有 brain (depth 0) 能从 worker pool delegate，worker (depth > 0) 只能动态 spawn**——刻意限制 worker 不知道 pool 里有哪些静态角色，避免策略复杂化。

### Task 执行

```python
@dataclass(frozen=True)
class Task:
    task_id: str; instruction: str
    context: tuple[str, ...] = ()
    max_turns: int = 20; timeout: int = 1800
    max_output_tokens: int = 2000
    metadata: dict = field(default_factory=dict)

async def _run_worker(agent, task, tracker) -> TaskResult:
    # 1. context 里每条尝试 json.loads 解 {role,content}，否则包 user message
    # 2. instruction 追加为最后一条 user message
    # 3. asyncio.timeout(task.timeout) 包住 agent.run()
    # 4. 超时→ 返回 TaskResult(status="cancelled", error="timeout")
    # 5. CancelledError → 返回 status="cancelled"
    # 6. 其他异常 → 返回 status="failed", error=str(exc)
    # 7. 成功 → 截断 output 到 max_output_tokens*4 chars，提取 work_summary
```

`_extract_work_summary(messages)`：纯逻辑（不调 LLM），遍历 assistant message 收集最后 5 个 tool_use 名字 + 最后一条 assistant 文本：

```
工具调用（最近 3 次）: Read, Grep, Write
最后输出: 审查完毕。发现 2 个 SQL 注入风险...
```

### Event Bubble 机制

```python
def _setup_event_bubble(worker_agent, orchestrator, worker_name, task_id, depth) -> Callable[[], None]:
    worker_bus = worker_agent._event_bus
    orchestrator_bus = orchestrator._event_bus
    
    def _bubble(event: Event) -> None:
        try:
            wrapped = WorkerEvent(worker_name, task_id, depth, inner=event)
            orchestrator_bus.emit(wrapped)
        except Exception:
            logger.exception("bubble error ...")
    
    worker_bus.subscribe_all(_bubble)  # 订阅全部 16 种事件类型
    
    def _teardown():
        for event_type in _ALL_EVENT_TYPES:
            worker_bus.unsubscribe(event_type, _bubble)
    
    return _teardown
```

每个 worker 产生的 event 被**原封装包装**为 `WorkerEvent(worker_name, task_id, depth, inner=原 event)` 发射到 orchestrator 的 bus——orchestrator Observer 一个订阅就能看到全部层级的事件流，保留原事件结构的同时附加"来自哪个 worker/任务/深度"的定位信息。

`spawn_worker`/`delegate_task` 的 finally 段会 `teardown()` 取消订阅，防止 handler 累积泄漏。

### Semaphore 并发控制

```python
self._semaphore = asyncio.Semaphore(max_concurrent_workers)  # default 5

async def _run_with_semaphore():
    async with orchestrator._semaphore:
        return await _run_worker(worker_agent, task, tracker)

asyncio_task = asyncio.create_task(_run_with_semaphore())
tracker.set_asyncio_task(task_id, asyncio_task)
```

**注意**：`track(task)` 先于 `set_asyncio_task` 调用——意味着**任务在等信号量时已可被 cancel_task**（tracker 里有 task 但还没 asyncio_task）。这种场景 `tracker.cancel(task_id)` 返回 False（`asyncio_task is None → no-op`）——设计选择是"等待队列里的任务不支持取消"。

### TaskTracker 的职责

- `_tasks: dict[task_id, Task]` — 所有注册过的任务
- `_results: dict[task_id, TaskResult]` — 完成的
- `_asyncio_tasks: dict[task_id, asyncio.Task]` — 正在跑的
- `_metadata: dict[task_id, {worker_name, instruction}]` — summary 用

`cleanup(max_completed=1000)`：保留最近 1000 个完成任务，LRU 淘汰——防止长期运行的 orchestrator 内存无限增长。

### 关键代码路径

- `neoagent/multi/orchestrator.py:36-118` — Orchestrator 构造 + 5 工具注册 + register_worker/load_workers/register_tool
- `neoagent/multi/worker.py:15-24` — WorkerCard frozen dataclass
- `neoagent/multi/worker.py:74-143` — `_create_worker_agent` 构造
- `neoagent/multi/task.py:18-62` — Task/TokenUsage/TaskResult
- `neoagent/multi/task.py:95-245` — TaskTracker
- `neoagent/multi/task.py:255-424` — `_extract_work_summary` + `_run_worker` 执行 + 异常分支
- `neoagent/multi/events.py:18-63` — `_setup_event_bubble`
- `neoagent/multi/tools/spawn_worker.py:56-128` — spawn 流程
- `neoagent/multi/tools/delegate_task.py:63-148` — delegate 流程
- `neoagent/multi/parser.py:10-80` — .md frontmatter 解析

## 设计亮点

- **WorkerCard 作为 frozen dataclass 从 .md 文件加载**：让 worker 定义在版本控制里、像 Claude Code 的 `.claude/commands/` 或 Cursor 的 `.cursorrules` 那样——不需要在代码里写一长串 register 调用。`load_workers(directory)` 批量加载。
- **Brain 和 Worker 都是 NeoAgent**：递归结构简洁，没有 "Orchestrator 特有的类型" 抽象泄漏；WorkerCard 只是 brain 做选择时的"信息卡"，实际执行用的仍是 NeoAgent。
- **Tool pool 独立于 tool registry**：`orchestrator._tool_pool` 是全量可用工具，brain 不直接访问（通过 list_workers 等工具列出），worker 按 `card.tools` 白名单领取 `copy.copy(tool)` 实例——权限最小化 + 实例隔离（一个 worker 改 tool 内部状态不影响其他 worker）。
- **Event bubble 层层累积 depth**：可以画出完整的 agent 任务树——`WorkerEvent(depth=0, inner=WorkerEvent(depth=1, inner=ToolCallEvent(...)))` 的嵌套结构保留了层级信息，Observer 可以按 depth 缩进输出。
- **`_create_worker_agent` 不暴露 DelegateTaskTool 给 worker**：worker 不能知道 pool 里有哪些 worker，避免 "worker 间互相 delegate" 的策略复杂化——保持严格的树形层级。
- **`spawn_worker` 的 md_definition 直接是 LLM 生成**：LLM 可以临时发明角色（"需要一个 Postgres 数据库专家"），不需要预先注册。这是"自我拆解任务"的激进设计。
- **`_run_with_semaphore` + track 先于 set_asyncio_task 的顺序**：保证在信号量满时新 task 能在 tracker 里显示为 "active" 而不是凭空消失。虽然此时 cancel_task 拿不到 asyncio.Task 无法真 cancel，但至少 list_tasks 看得到。
- **TaskResult 包含 work_summary**：不依赖 LLM 就能产出"这个 worker 做了啥"的摘要（工具调用 + 最后输出），brain 看到后能判断是否要 re-dispatch。
- **`copy.copy(tool)` 浅拷贝**：避免把同一 tool 实例注册到多个 worker 的 ToolRegistry 造成共享状态——这是"多个 worker 各自持有工具副本"的隔离设计。

## 局限性

- **`copy.copy` 不深拷贝 tool 内部状态**：如果 tool 包含 dict/list 可变字段（如 MemoryStore），多 worker 会共享底层对象——潜在状态污染。
- **静态 worker 不支持 MCP 工具**：`card.tools` 是工具名字符串白名单，只能匹配 `_tool_pool` 里注册的工具；orchestrator 的 brain 如果接了 MCP，worker 无法继承——需要手动把 MCPTool 也注册到 `_tool_pool`。
- **brain 和 worker 共享 api_key**：`worker_config.api_key = orch_config.api_key` 硬编码——无法给敏感 worker 用独立 API key（细粒度计费 / 权限隔离缺失）。
- **Worker context_budget 继承 orchestrator 的同一值**：不能按 worker 类型差异化，reviewer 和 code-writer 用相同 budget——不符合"简单任务配小 budget"的优化直觉。
- **Event bubble 注册全部 16 种事件**：即便 orchestrator 只关心 Task*Event 和 WorkerEvent，仍要 subscribe_all——bus 内存消耗 + handler 开销大。
- **max_output_tokens 用 4 chars/token 估算**：和 ContextCompressor 的 tiktoken 精确估算不一致——worker 返回的 output 被粗暴按 `max_output_tokens * 4` 字符截断，可能切到 UTF-8 多字节中间。
- **Cancel 时等待队列中的 task 无法取消**：`cancel(task_id)` 在 `asyncio_task is None` 时返回 False——但 user 不知道"刚 delegate 的任务在等 semaphore"，UI 可能提示"取消失败"让用户困惑。
- **没有 CostTracker**：OpenHarness 和 Hermes 都有 per-task token/cost 累计用于 budget 控制，neoagent 的 `TokenUsage` 只是信息回传，不在 brain 层聚合——10 个 worker 同时跑各自消耗 10K tokens，brain 不会主动停。
- **orchestrator.run() 只返回 brain.chat() 的最终字符串**：没有树形结构返回（谁 dispatch 了什么、每个 worker 的 TaskResult）——观察全貌必须订阅 event bus，调用方 API 单薄。
- **.md worker 定义无 schema 验证**：YAML frontmatter 里 `tools: [Read, Grep]` 如果 `Read` 名字拼错，`_create_worker_agent` 静默 skip（只 log debug）——worker 运行时会少工具但不报错，可能产生无法解释的行为。
- **无 cycle detection**：理论上可以构造 "worker A spawn worker B, worker B spawn worker A" 循环——`max_depth` 是唯一的安全阀，递归只被深度限制而非环检测限制。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/multi/orchestrator.py`, `worker.py`, `task.py`, `events.py`, `parser.py`, `tools/spawn_worker.py`, `tools/delegate_task.py`, `tools/cancel_task.py`
