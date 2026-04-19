---
title: "evaluation-observability — neoagent"
category: L2
parent: "[[evaluation-observability]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: evaluation-observability
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的可观测 + 评估体系由三层组成：(1) **`EventBus` + 16 种 frozen dataclass 事件**（`events.py`）作为整个框架的唯一观测通道；(2) **三个 Subscriber 适配器**：`ObserverSubscriber`（日志化）、`MetricsCollector`（per-turn 度量）、`UsageTracker`（per-model token 累计），订阅-发布模式完全解耦业务逻辑和观测；(3) **`EvalRunner` 批量评估框架**（`eval/runner.py`）：`EvalCase(name, messages, assertion, max_turns)` 批量跑，每 case 独立 MetricsCollector，try/finally 保证 unsubscribe 不泄漏 handler，异常不中断剩余用例。Observer 采用"conversation-style"格式（按 SYSTEM/USER/ASSISTANT/TOOL 四个角色着色），既给 CLI 使用又写日志文件。

## 架构分析

### EventBus 最简实现（25 行核心）

```python
class EventBus:
    def __init__(self):
        self._handlers: dict[type[Event], list[Handler]] = {}
    
    def subscribe(self, event_type: type[T], handler):
        self._handlers.setdefault(event_type, []).append(handler)
    
    def unsubscribe(self, event_type, handler):
        try: self._handlers[event_type].remove(handler)
        except (KeyError, ValueError): pass
    
    def emit(self, event: Event):
        for handler in list(self._handlers.get(type(event), [])):
            try: handler(event)
            except Exception:
                logger.exception("EventBus handler error for %s", type(event).__name__)
    
    def subscribe_all(self, handler):
        for event_type in _ALL_EVENT_TYPES:
            self.subscribe(event_type, handler)
```

特征：
- **同步 dispatch**：handler 调用不 await（handler 必须是普通函数）
- **`list(self._handlers.get(...))`** 创建副本迭代——防止 handler 在 emit 过程中 subscribe/unsubscribe 导致 `dict changed size` 错误
- **handler 异常不中断其他 handler**：log.exception + continue，一个坏 subscriber 不毒化观测流
- **无 event filtering**：handler 接受整条 event 自己判断——框架不做属性匹配

### 16 种 Event 类型清单

所有事件都是 `@dataclass(frozen=True)`，按域分组：

| 域 | 事件 | 触发时机 |
|----|------|---------|
| Provider | `ProviderRequestEvent(system, messages, tools, turn)` | QueryLoop 调 provider.create() 前 |
| Provider | `ProviderResponseEvent(content, stop_reason, input_tokens, output_tokens, turn)` | provider.create() 返回后 |
| Tool | `ToolCallEvent(name, input_data: MappingProxyType, call_id)` | ToolExecutor._run_one 开始 |
| Tool | `ToolResultEvent(name, call_id, output, is_error)` | ToolExecutor._run_one 返回前 |
| Compress | `CompressCheckEvent(msg_tokens, tool_tokens, budget, should_compress)` | 每轮 should_compress 判断后 |
| Compress | `CompressDoneEvent(summary, previous_summary)` | LLM 摘要成功 |
| Compress | `CompressFallbackEvent(reason)` | LLM 失败降级截断 |
| Memory | `MemoryExtractEvent(triggered, tool_calls, token_delta, items_stored, filenames)` | end_turn 后 maybe_extract |
| Skill | `SkillChangeEvent(name, active)` | **未发射**（TODO v3.2） |
| Turn | `TurnCompleteEvent(turn_index, stop_reason, tool_call_count)` | 每轮结束 |
| ToolResult lifecycle | `ToolResultFreedEvent(tool_use_id, tool_name, size, preview, reason)` | auto_free_after 或 manual free |
| ToolResult lifecycle | `ToolResultRecalledEvent(tool_use_id, tool_name)` | **未发射** |
| Session | `SessionResumeWarningEvent(session_id, reason, details)` | resume 校验发现 workspace_missing / stale_session |
| Multi-agent | `WorkerEvent(worker_name, task_id, depth, inner: Event)` | worker event bus 事件冒泡 |
| Multi-agent | `TaskDispatchEvent(task_id, worker_name, instruction, depth)` | spawn/delegate 调度时 |
| Multi-agent | `TaskCompleteEvent(task_id, worker_name, status, turns_completed, usage)` | task 完成时 |

### Subscriber 一：Observer + ObserverSubscriber

`Observer`（`observe.py` 269 行）是"conversation-style" CLI 输出：

- 角色着色：`[SYSTEM]` 青、`[USER]` 绿、`[ASSISTANT]` 黄、`[TOOL]` 蓝（ANSI）
- `[COMPRESS]`/`[MEMORY]`/`[FREED]`/`[SKILL]` 各有专用行
- 时间戳 `HH:MM:SS.mmm`、`_truncate(text, max_len=500)` 截断显示 + 日志写全文
- `log_dir` 可选 → 开启文件日志：`{timestamp}-{name}.log`，用 `_strip_ansi` 移除颜色码
- `name` 参数让 multi-agent 每 worker 带前缀 `[worker-reviewer]`

`ObserverSubscriber`（`observe_subscriber.py` 87 行）是适配层——把 10 种 EventBus 事件映射到 Observer 的 11 个 `on_*` 方法：

```python
class ObserverSubscriber:
    def attach(self, bus: EventBus):
        pairs = [
            (ProviderRequestEvent, self._on_provider_request),
            (ProviderResponseEvent, self._on_provider_response),
            (ToolCallEvent, self._on_tool_call),
            (ToolResultEvent, self._on_tool_result),
            (CompressCheckEvent, self._on_compress_check),
            (CompressDoneEvent, self._on_compress_done),
            (CompressFallbackEvent, self._on_compress_fallback),
            (MemoryExtractEvent, self._on_memory_extract),
            (ToolResultFreedEvent, self._on_tool_result_freed),
        ]
        for event_type, handler in pairs:
            bus.subscribe(event_type, handler)
            self._handlers.append((event_type, handler))
    
    def detach(self, bus):
        for event_type, handler in self._handlers:
            bus.unsubscribe(event_type, handler)
        self._handlers.clear()
```

`NeoAgent.enable_logging(log_dir, console)` 一行调用建立 subscriber + Observer + attach：

```python
def enable_logging(self, log_dir=None, console=True) -> Observer:
    # detach 旧的防止泄漏
    if self._observer_subscriber:
        self._observer_subscriber.detach(self._event_bus)
    if self._observer:
        self._observer.close()
    
    log_dir = log_dir or Path.cwd() / "logs"
    observer = Observer(log_dir=log_dir, console=console)
    subscriber = ObserverSubscriber(observer)
    subscriber.attach(self._event_bus)
    self._observer_subscriber = subscriber
    self._observer = observer
    return observer
```

### Subscriber 二：MetricsCollector

`eval/metrics.py` 的 `MetricsCollector` 订阅 4 个事件（Provider Request/Response、ToolCall、TurnComplete）：

```python
@dataclass
class TurnMetrics:
    turn_index: int
    input_tokens: int
    output_tokens: int
    latency_ms: float         # response_time - request_time
    tool_call_count: int
    tool_names: list[str]

class MetricsCollector:
    def _on_provider_request(self, event):
        self._get_or_create_pending(event.turn)["request_time"] = time.monotonic()
    
    def _on_provider_response(self, event):
        p = self._get_or_create_pending(event.turn)
        p["response_time"] = time.monotonic()
        p["input_tokens"] += event.input_tokens
        p["output_tokens"] += event.output_tokens
    
    def _on_tool_call(self, event):
        # ToolCallEvent 无 turn 字段，assign 给 max pending turn
        turn = max(self._pending.keys()) if self._pending else 0
        p = self._get_or_create_pending(turn)
        p["tool_call_count"] += 1
        p["tool_names"].append(event.name)
    
    def _on_turn_complete(self, event):
        pending = self._pending.pop(event.turn_index, None)
        ...  # 构造 TurnMetrics 加入 _completed_turns
```

`SessionMetrics` 是 `TurnMetrics` 的 aggregator，提供 `total_input_tokens` / `total_output_tokens` / `duration_ms` / `tool_calls` 聚合 property。

`enable_metrics()` 一行调用建立：

```python
def enable_metrics(self) -> MetricsCollector:
    collector = MetricsCollector()
    self._event_bus.subscribe(ProviderRequestEvent, collector._on_provider_request)
    self._event_bus.subscribe(ProviderResponseEvent, collector._on_provider_response)
    self._event_bus.subscribe(ToolCallEvent, collector._on_tool_call)
    self._event_bus.subscribe(TurnCompleteEvent, collector._on_turn_complete)
    return collector
```

### Subscriber 三：UsageTracker

`eval/usage.py` 最简单——订阅 ProviderResponseEvent，按 `_current_model`（由调用方设置）累加 input/output tokens：

```python
@dataclass
class ModelUsage:
    model: str
    input_tokens: int
    output_tokens: int
    request_count: int

class UsageTracker:
    def _handle_response(self, event):
        model = self._current_model
        if model not in self._per_model:
            self._per_model[model] = ModelUsage(model=model)
        u = self._per_model[model]
        u.input_tokens += event.input_tokens
        u.output_tokens += event.output_tokens
        u.request_count += 1
```

多 provider / 多模型场景下能分别统计。局限：`_current_model` 在 `enable_usage_tracking` 时 snapshot，agent 运行时切换 model 不自动跟随。

### EvalRunner 批量评估

```python
@dataclass
class EvalCase:
    name: str
    messages: list[Message]
    assertion: Callable[[ConversationResult], bool | Awaitable[bool]]
    max_turns: int | None = None
    metadata: dict = field(default_factory=dict)

@dataclass
class EvalCaseResult:
    name: str
    passed: bool
    result: ConversationResult | None
    error: str | None
    metrics: SessionMetrics

@dataclass
class EvalReport:
    total: int
    passed: int
    failed: int
    cases: list[EvalCaseResult]
    
    @property
    def pass_rate(self) -> float: ...

class EvalRunner:
    async def _run_case(self, case):
        collector = MetricsCollector()
        bus = self._agent.event_bus
        bus.subscribe(ProviderRequestEvent, collector._on_provider_request)
        # ... 4 handlers
        
        try:
            try:
                conv_result = await self._agent.run(messages=list(case.messages), max_turns=case.max_turns)
            except Exception as exc:
                error = f"agent raised: {type(exc).__name__}: {exc}"
                return EvalCaseResult(name=case.name, passed=False, ..., metrics=collector.get_session_metrics())
            
            try:
                assertion_result = case.assertion(conv_result)
                if inspect.isawaitable(assertion_result):
                    assertion_result = await assertion_result
                passed = bool(assertion_result)
            except Exception as exc:
                error = f"assertion raised: {type(exc).__name__}: {exc}"
                passed = False
        finally:
            # 4 次 unsubscribe 成对清理
            bus.unsubscribe(ProviderRequestEvent, collector._on_provider_request)
            # ...
        
        return EvalCaseResult(...)
```

关键设计：
- **每 case 独立 MetricsCollector**：case 之间 metrics 隔离，不跨 case 污染
- **双重 try 分层捕获**：外层捕获 agent 异常、内层捕获 assertion 异常——两类错误分别归因
- **`inspect.isawaitable` 支持 async assertion**：允许 `async def my_assertion(result): ...`
- **finally unsubscribe**：即使 assertion 抛异常也保证 4 个 handler 清理

### 多 Agent 的 Event Bubble

（详见 multi-agent L2）
`_setup_event_bubble(worker_agent, orchestrator, worker_name, task_id, depth)` 订阅 worker bus 全 16 种事件，包成 `WorkerEvent(worker_name, task_id, depth, inner)` 发射到 orchestrator bus——Observer 一个 attach 就能看到全部层级的事件流，嵌套 `WorkerEvent(depth=0, inner=WorkerEvent(depth=1, inner=ToolCallEvent(...)))` 保留完整血缘。

### 关键代码路径

- `neoagent/events.py:180-205` — EventBus 核心
- `neoagent/events.py:13-176` — 16 种 Event dataclass + `_ALL_EVENT_TYPES` 列表
- `neoagent/observe.py:54-269` — Observer CLI 格式化
- `neoagent/observe_subscriber.py:14-87` — ObserverSubscriber 适配器
- `neoagent/eval/metrics.py:48-149` — MetricsCollector
- `neoagent/eval/runner.py:58-149` — EvalRunner
- `neoagent/eval/usage.py:17-45` — UsageTracker
- `neoagent/agent.py:206-288` — `enable_logging/enable_metrics/enable_usage_tracking`

## 设计亮点

- **Frozen dataclass event + ContextVar-free 设计**：event 不可变，不依赖 ContextVar 传递——handler 在不同 thread/task 里看到的都是同一个不可变 snapshot，适合多 agent / FastAPI 并发。
- **Subscribe-publish 解耦**：业务逻辑（QueryLoop / ToolExecutor / MemoryManager）只 emit，不关心谁在听——加新观测点（metrics、debug log、遥测上报）不需要改业务代码。
- **多 Subscriber 独立注册**：Observer、MetricsCollector、UsageTracker 三套 subscriber 并行工作互不干扰；用户可以按需组合 `enable_logging()` + `enable_metrics()` + `enable_usage_tracking()`。
- **`enable_logging` 的幂等 detach**：多次调用不泄漏——先 detach 旧 subscriber + close 旧 observer 再创建新的。
- **EvalRunner 的异常归因**：双重 try / catch 让 `EvalCaseResult.error` 明确区分 "agent raised" vs "assertion raised"——用户看到失败马上知道是 agent 功能问题还是 assertion 写错。
- **Per-case MetricsCollector 隔离**：eval 批次中每个 case 的 token 消耗、latency、tool 调用各自独立统计——方便排查"是哪个 case 最贵"。
- **`subscribe_all` + WorkerEvent 嵌套**：multi-agent 场景用一招实现全链路观测——orchestrator 订阅 WorkerEvent 就能看到 worker 链上的任何事件。
- **Observer 格式化的细节**：`_truncate(text, 500)` 到屏幕但写文件 full body（`_write(console_line, full=file_line)`）——debug 时看全文，CLI 看摘要，兼顾两种使用体验。
- **`NeoAgent.event_bus` property 公开**：外部 subscriber（如 prometheus exporter、OTEL span 适配器）可以直接订阅——没有"观测权限"门禁。

## 局限性

- **SkillChangeEvent / ToolResultRecalledEvent 定义了但未发射**：源码注释"TODO v3.2"——skill 生命周期、recall 行为目前无法被观察。
- **`MemoryExtractEvent.filenames = ()` 未填充**：loop.py:249 注释明写——Observer 只知道"存了 N 条"，不知道"存了哪些文件"。
- **同步 handler 限制**：async handler 调用返回 coroutine 对象不执行——multi-agent event bubble 被迫写成 sync，外部集成 OTEL 等 async-heavy 系统需要自己桥接到 task queue。
- **无 subscription filter**：想要"只听 tool_name='write_file' 的 ToolCallEvent"必须 handler 自己 filter；大型部署可能有数百 subscriber，每个 event 全部分发开销大。
- **ToolCallEvent 无 turn 字段**：MetricsCollector 只能"assign 给 max pending turn"启发式 —— 一轮未完但已 emit ProviderRequestEvent 时工具调用归属正确，但跨异步边界（如 hook 改了 call）可能归错。
- **`UsageTracker._current_model` 静态 snapshot**：agent 运行时 provider 切换模型（例如 fallback）时，累计仍归到原 model——和 Hermes Agent 的 session-scoped model tracking 相比粒度弱。
- **无内置 OpenTelemetry 支持**：AgentScope 的 tool-system 有 `@trace_toolkit` 装饰器自动产生 OTEL span，neoagent 需要用户自己写 subscriber 桥接。
- **EvalRunner 只支持 sequential 跑**：`for case in cases: await self._run_case(case)`——无 `asyncio.gather` 批量并发，大批 case 评估慢。
- **无 baseline / regression 比较**：EvalReport 只输出当前 pass rate，不和上次结果比；regression detection 要用户自己保存历史 report。
- **Observer `_truncate(text, 500)` 硬编码**：长 tool result / long prompt 强制截断——调试长输出时只能从日志文件找。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/events.py`, `neoagent/observe.py`, `neoagent/observe_subscriber.py`, `neoagent/eval/{metrics,runner,usage}.py`, `neoagent/agent.py:206-288`, `neoagent/multi/events.py`
