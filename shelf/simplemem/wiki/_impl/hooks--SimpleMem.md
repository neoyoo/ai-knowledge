---
title: "hooks——SimpleMem"
category: L2
parent: "[[hooks]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: hooks
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

SimpleMem 的 hooks 系统在 `cross/hooks.py` 中实现，是 cross/ 层与 Agent 框架集成的核心扩展点。通过抽象基类 `SessionHooks` 定义 agent 生命周期钩子接口，框架开发者实现此 ABC 即可将 SimpleMem-Cross 集成到任何 agent 运行时。`cross/context_injector.py` 是 hooks 系统的伴生组件，负责在 `on_session_start` 阶段将跨会话 context 注入 agent 的 system prompt。

## 架构分析

### SessionHooks ABC（hook 接口定义）

```python
class SessionHooks(ABC):
    """Agent 生命周期钩子抽象接口。
    
    生命周期顺序：
    SessionStart → (UserMessage | ToolUse)* → SessionStop → SessionEnd
    """
    
    @abstractmethod
    async def on_session_start(tenant_id, content_session_id, project, user_prompt) -> HookResult
    
    @abstractmethod
    async def on_user_message(memory_session_id, content) -> HookResult
    
    @abstractmethod
    async def on_assistant_message(memory_session_id, content) -> HookResult
    
    @abstractmethod
    async def on_tool_use(memory_session_id, tool_name, tool_input, tool_output) -> HookResult
    
    @abstractmethod
    async def on_session_stop(memory_session_id) -> HookResult
    
    @abstractmethod
    async def on_session_end(memory_session_id) -> HookResult
```

### HookResult 统一返回类型

```python
class HookResult(BaseModel):
    context_bundle: Optional[ContextBundle] = None       # session_start 时返回历史 context
    finalization_report: Optional[FinalizationReport] = None  # session_stop 时返回摘要报告
    events_recorded: int = Field(default=0, ge=0)        # 本次 hook 记录的事件数
```

### ContextInjector（context_injector.py）

在 `on_session_start` hook 中被调用，职责：
1. 从 SQLite 检索历史 session summaries（按项目和租户过滤）
2. 从 LanceDB 做语义搜索（基于 user_prompt）
3. 按 token 预算截断，组装 `ContextBundle`
4. 生成 formatted_context 字符串，可直接拼入 system prompt

### 事件收集器（collectors.py 中的 EventCollector）

`EventCollector` 是 hooks 调用的写后缓冲器：
- 每次 hook 调用（on_user_message / on_tool_use）将事件追加到内存 buffer
- `session_stop` 时批量 flush 到 SQLite
- 同时触发 `ObservationExtractor.extract()` 从事件流中提取结构化观察

## 关键代码路径

### Hook 调用链（on_session_start 完整路径）

```python
# 某 Agent 框架调用（如 Claude Code 的 stop hook）
hooks = SimpleMemCrossHooks(orchestrator)
result = await hooks.on_session_start(
    tenant_id="user-123",
    content_session_id="session-456",
    project="/my/project",
    user_prompt="Fix the login bug"
)

# hooks.py → orchestrator.py
async def on_session_start(self, tenant_id, content_session_id, project, user_prompt):
    result = await self._orchestrator.session_start(
        tenant_id=tenant_id,
        content_session_id=content_session_id,
        project=project,
        user_prompt=user_prompt,
    )
    return HookResult(
        context_bundle=result.context_bundle,
        events_recorded=0,
    )

# orchestrator.py → context_injector.py
def session_start(self, ...):
    with self._lock:
        session = MemorySession(memory_session_id=uuid4(), ...)
        self._active_sessions[session.memory_session_id] = session
    bundle = self._context_injector.build_context_bundle(
        tenant_id=tenant_id, user_prompt=user_prompt, token_budget=config.CONTEXT_TOKEN_BUDGET
    )
    return HookResult(context_bundle=bundle)
```

### Hook 调用链（on_session_stop → finalization）

```python
result = await hooks.on_session_stop(memory_session_id="xyz")

# hooks.py → orchestrator.session_stop()
# orchestrator.py → session_manager.finalize_session()
async def finalize_session(self, memory_session_id):
    session = self._active_sessions[memory_session_id]
    
    # EventCollector → flush to SQLite
    events = session.event_collector.flush()
    self.storage.save_events(events)
    
    # ObservationExtractor → LLM → CrossObservation[]
    observations = self.obs_extractor.extract(events)
    self.storage.save_observations(observations)
    self.vector_store.add_observations(observations)
    
    # LLM summarize session
    summary = self.llm_client.summarize(events)
    self.storage.save_summary(summary)
    
    return FinalizationReport(
        observations_count=len(observations),
        summary_generated=bool(summary),
        entries_stored=len(observations)
    )
```

### ObservationExtractor（从事件流提取结构化观察）

```python
# cross/collectors.py
class ObservationExtractor:
    def extract(self, events: List[SessionEvent]) -> List[CrossObservation]:
        # 合并 events 为文本
        event_text = format_events(events)
        # LLM 提取结构化观察
        prompt = build_observation_prompt(event_text)
        response = llm_client.chat_completion([...], temperature=0.1)
        observations = parse_observations(response)
        # 每条 observation 包含:
        # type: decision/bugfix/feature/refactor/discovery/change
        # title: 简短标题
        # narrative: 详细描述
        return observations
```

## 设计亮点

1. **纯抽象接口设计**：`SessionHooks` 是纯 ABC，不依赖任何具体 agent 框架，Claude Code、LangGraph、AutoGen 等框架只需实现 6 个方法即可集成。

2. **异步优先**：所有 hook 方法均为 `async`，配合 `_await_if_coro()` 工具函数实现同步/异步透明适配，不强制 orchestrator 必须异步。

3. **事件类型化（EventKind 枚举）**：事件不是原始字符串，而是 `message/tool_use/file_change/note/system` 枚举类型，使 ObservationExtractor 能做类型感知的提取，提高 observation 质量。

4. **FinalizationReport 可观测性**：`on_session_stop` 返回的 report 包含 observations_count/summary_generated/entries_stored，框架可记录这些指标用于监控记忆系统健康状态。

5. **Redaction Level**：`SessionEvent.redaction_level` 支持 none/partial/full 三级脱敏，敏感事件（如密码输入）可标记为 full 不参与 observation 提取。

## 局限性

1. **无同步实现**：`SessionHooks` ABC 全部是 async 方法，同步 agent 框架（如简单 Python 脚本）必须在 event loop 中调用，增加集成复杂度。

2. **Finalization 是在线 LLM 调用**：`on_session_stop` 触发 ObservationExtractor 需要 LLM 调用，若 session 包含大量事件，finalization 耗时可能超过 agent 框架的 hook 超时限制。

3. **无 Hook 优先级或链式调用**：只能注册一个 `SessionHooks` 实现，无法串联多个 hook（如日志 hook + 记忆 hook），扩展性受限。

4. **on_file_change 未完整实现**：`EventKind.file_change` 类型存在但 hooks.py 中未暴露 `on_file_change` 方法，file change 事件只能通过 `on_user_message` 或 note 类型变通记录。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
