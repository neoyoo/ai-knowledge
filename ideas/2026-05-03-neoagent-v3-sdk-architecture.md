---
name: neoagent v3 sdk architecture
description: neoagent v3 重写版 SDK 工程骨架。以 context protocol 作为认知模型，以 ai-knowledge 概念模块作为 SDK 运行时结构。
type: architecture
status: inbox
date: 2026-05-03
relates_to:
  - ideas/2026-05-02-neoagent-context-protocol-v3.md
  - ideas/2026-05-03-neoagent-llm-context-only-example.md
  - wiki/_index.md
  - docs/superpowers/specs/2026-04-14-neoagent-sdk-skill-design.md
---

# neoagent v3 SDK Architecture

## 1. 顶层原则

neoagent v3 的重写不以旧 neoagent 包结构为架构约束，而以两个上层设计作为边界：

```text
Context protocol 决定 agent 的认知模型。
ai-knowledge 模块体系决定 SDK 的工程骨架。
旧 neoagent 代码只作为局部实现参考，不作为架构约束。
```

这意味着：

- LLM 每轮看到什么、怎么维护 working state、怎么压缩和召回，由 context protocol 决定。
- SDK 有哪些运行时模块、模块之间怎么协作，由 ai-knowledge 的概念地图决定。
- 旧代码中可复用的 provider、tool、MCP、配置、测试经验可以搬，但旧 prompt / working memory / compression 抽象不能直接搬。

---

## 2. ai-knowledge → neoagent v3 模块映射

| ai-knowledge 概念 | v3 模块 | 责任 |
|---|---|---|
| `query-loop` | `runtime/loop.py` | Agent 主循环和 turn 调度 |
| `runtime-state` | `runtime/session.py`, `runtime/turn.py` | session、turn、运行状态生命周期 |
| `context-management` | `context/`, `messages/`, `compression/`, `recall/` | 上下文投影、窗口、压缩、恢复 |
| `prompt-system` | `context/renderer.py`, `context/projection.py` | 渲染 LLM 可见 context，不做旧式 PromptBuilder 拼接 |
| `tool-system` | `capabilities/tools.py`, `capabilities/executor.py` | 工具注册、权限、执行、结果回写 |
| `memory-system` | `memory/` | 跨 session memory 的提取、存储、召回 |
| `mcp-skills` | `capabilities/mcp.py`, `capabilities/skills.py` | MCP 连接和 skill 加载 |
| `multi-agent` | `multi/` | subagent 派发、隔离、结果回收 |
| `hooks` | `runtime/event_bus.py`, `hooks/` | 事件驱动扩展点 |
| `session-recovery` | `persistence/`, `runtime/session.py` | 断点恢复和持久化 |
| `evaluation-observability` | `observability/`, `eval/` | 事件日志、trace、Langfuse/OTel、评估 |
| `channel-remote` | `channels/` | CLI / HTTP / 服务化入口 |
| `sandbox-isolation` | `policies/security.py`, `capabilities/executor.py` | 工具权限和执行隔离 |

---

## 3. 目标包结构

```text
neoagent/
  runtime/
    agent.py
    loop.py
    request_builder.py
    session.py
    turn.py
    event_bus.py

  context/
    state.py
    schema.py
    renderer.py
    projection.py
    runtime.py
    chapter.py

  messages/
    types.py
    store.py
    window.py
    runtime.py

  compression/
    evictor.py
    compressor.py
    index.py
    runtime.py

  recall/
    runtime.py

  capabilities/
    registry.py
    tools.py
    executor.py
    context_tools.py
    skills.py
    mcp.py

  providers/
    base.py
    openai.py
    anthropic.py
    stream.py

  memory/
    extractor.py
    retriever.py
    store.py
    runtime.py

  observability/
    events.py
    traces.py
    langfuse.py
    otel.py

  persistence/
    base.py
    memory.py
    sqlite.py
    filesystem.py

  policies/
    budget.py
    security.py
    tool_policy.py

  multi/
    orchestrator.py
    worker.py
    result.py

  channels/
    base.py
    cli.py
    http.py

  eval/
    runner.py
    cases.py
    metrics.py
```

---

## 4. 核心数据流

```text
User input
  ↓
MessageRuntime.append_user()
  ↓
AgentLoop
  ↓
ContextRuntime.prepare_for_request()
  ├─ apply pending context state
  ├─ maybe compress active messages
  ├─ maybe inject recalled context
  └─ render LLM-visible context
  ↓
RequestBuilder.build()
  ├─ system = ContextRuntime.render()
  ├─ messages = MessageRuntime.materialize_active()
  └─ tools = CapabilityRuntime.tool_specs()
  ↓
ProviderRuntime.complete()
  ↓
MessageRuntime.append_assistant()
  ↓
CapabilityRuntime.execute(tool_calls)
  ├─ ContextToolExecutor → ContextRuntime
  ├─ ExternalToolExecutor → files/db/shell/http
  └─ Skill/MCP executors
  ↓
MessageRuntime.append_tool_results()
  ↓
loop until assistant final response
```

`AgentLoop` 只做调度，不直接拼 prompt、不直接改 context、不直接执行具体工具。

---

## 5. Context Runtime

Context Runtime 是 agent 的认知模型实现。

### 5.1 责任

- 保存 `ContextState`。
- 管理 `WorkingStateSchema` 和 `WorkingState`。
- 执行 context protocol tools。
- 渲染 LLM 可见上下文。
- 管理 compressed history、inherited state、memory context 的 projection。
- 默认不向 prompt 暴露 runtime metadata。

### 5.2 关键对象

```text
ContextState
WorkingStateSchema
WorkingState
WorkingStateField
CompressedSegment
MemoryContext
InheritedState
ContextProjection
```

### 5.3 默认可见 context sections

与 `neoagent LLM 可见上下文范文` 对齐：

```text
Runtime Contract
Capability Plane
Context Management Rules
Declared Working State Schema
Working State
Compressed History
Memory Context
```

### 5.4 Context tools

```text
declare_schema
update_state
extend_schema
start_chapter
recall_context
```

`read_state`、`abort_chapter`、`mark_important` 不作为默认 LLM 可见工具，可作为 debug/ops 能力后续添加。

### 5.5 不做什么

- 不存储 provider 原始 messages。
- 不执行外部工具。
- 不直接管理 session 持久化。
- 不把 `session_id`、`trace_id`、`message_id`、`compression_id` 等 runtime metadata 渲染进默认 prompt。

---

## 6. Message Runtime

Message Runtime 是 messages 真值源和 active window 管理器。

### 6.1 责任

- append-only 保存原始 messages。
- 维护 `ActiveWindow`。
- 保护 `tool_use` / `tool_result` 配对。
- 支持 recalled messages 的临时注入和自动剔除。
- 给 provider request materialize active messages。

### 6.2 关键对象

```text
Message
MessageRef
MessageStore
ActiveWindow
TemporaryMessage
```

### 6.3 压缩边界

压缩只从 `ActiveWindow` 移除 message refs，不删除 `MessageStore` 原文。

---

## 7. Compression + Recall

Compression 和 Recall 是 Message Runtime 与 Context Runtime 的桥。

### 7.1 压缩流程

```text
BudgetPolicy detects overflow
  ↓
Evictor selects contiguous message refs
  ↓
Compressor reads original messages from MessageStore
  ↓
CompressedSegment is created
  ↓
CompressionIndex maps seg handle to source message refs
  ↓
ActiveWindow removes selected refs
  ↓
ContextState.M3 appends segment
```

### 7.2 Recall 流程

```text
LLM calls recall_context(handle="seg_1")
  ↓
RecallRuntime looks up CompressionIndex
  ↓
MessageStore returns source messages
  ↓
MessageRuntime injects temporary recalled messages
  ↓
next request includes recalled content
  ↓
following request removes temporary recalled content
```

### 7.3 Compressor 类型

```text
RuleBasedCompressor      # 测试、fallback、确定性摘要
LLMCompressor            # 真实摘要
```

---

## 8. Capability Runtime

Capability Runtime 统一管理 tools、skills、MCP。

### 8.1 责任

- 注册工具。
- 暴露 provider tool schemas。
- 执行 tool calls。
- 应用 tool policy 和 security policy。
- 将 tool results 写回 Message Runtime。
- 将 context tool calls 路由到 Context Runtime。

### 8.2 工具分类

```text
Context tools      # declare_schema / update_state / extend_schema / start_chapter / recall_context
Builtin tools      # read_file / edit_file / run_shell / ask_user 等
MCP tools          # 来自 MCP server
Skill tool         # 加载 skill 指令
Subagent tool      # 派发子 agent
```

### 8.3 Skills

Skills 属于 capability plane，不属于 context projection。

参考 Claude Code 的方向：

- system 中只列 skill 摘要。
- 通过 `Skill` tool 加载具体 skill。
- 加载后的 skill 内容作为 meta message 注入。

---

## 9. Provider Runtime

Provider Runtime 不知道 context 内部细节。

### 9.1 输入

```text
ProviderRequest
  system: rendered context
  messages: active messages
  tools: provider tool schemas
```

### 9.2 责任

- 适配 OpenAI / Anthropic 等 provider。
- 处理 tool call schema。
- 处理 streaming。
- 返回标准化 `ProviderResponse`。
- 上报 usage 给 Observability Runtime。

---

## 10. Observability Runtime

Observability Runtime 是 runtime metadata 的归宿，不污染 prompt。

### 10.1 记录内容

```text
session_id
turn_id
message_id
trace_id
span_id
tool_call_id
schema_id
projection_id
compression_id
recall events
budget events
provider usage
tool execution events
```

### 10.2 输出

```text
Internal event log
Langfuse adapter
OTel adapter
Debug projection
Eval traces
```

---

## 11. Persistence

Persistence 给 runtime 提供可替换存储。

### 11.1 第一批实现

```text
MemoryPersistence
SQLitePersistence
FileSystemPersistence
```

### 11.2 存储对象

```text
sessions
turns
messages
context state
compressed segments
compression indexes
memory facts
observability events
```

---

## 12. Multi-Agent

Multi-Agent 是独立能力，不共享主 agent 的 context state。

### 12.1 原则

- 每个 subagent 有独立 `ContextRuntime`、`MessageRuntime`、`CapabilityRuntime`。
- 主 agent 只看到 subagent 的 tool result。
- 主 agent 想吸收 subagent 发现，必须显式 `update_state`。
- Subagent 权限不能超过父 agent。

### 12.2 不做什么

- 不允许 subagent 直接读写主 agent working state。
- 不默认共享 active messages。
- 不默认继承全部工具。

---

## 13. 旧 neoagent 代码复用规则

### 13.1 可以参考或搬运

- Provider API 调用细节。
- Tool calling schema 适配经验。
- MCP client 生命周期管理。
- Skill discovery 经验。
- 配置加载。
- 事件与观察者实现经验。
- 测试 fixtures。

### 13.2 不要搬运

- 旧 8 层 prompt renderer。
- 旧 working memory 字段模型。
- 旧 PromptBuilder 拼接抽象。
- 旧 compressed history schema。
- 旧 memory context 注入方式。
- 任何把 runtime metadata 直接渲染进 prompt 的逻辑。

---

## 14. 实施阶段

### Phase 1: Context + Messages 主链

- ContextState
- ContextRenderer
- Context tools
- MessageStore
- ActiveWindow
- RequestBuilder
- FakeProvider

验收：

- 能渲染 context-only 范文结构。
- 能通过 tools 更新 working state。
- 能生成 provider request。
- 默认 prompt 不暴露 runtime metadata。

### Phase 2: Compression + Recall

- BudgetPolicy
- Evictor
- RuleBasedCompressor
- CompressionIndex
- recall_context

验收：

- 旧 messages 从 active window 移除。
- 原文仍在 MessageStore。
- prompt 出现 `seg_1`。
- `recall_context("seg_1")` 能临时恢复原文。

### Phase 3: Providers + Tools

- OpenAI provider
- Anthropic provider
- ToolRegistry
- ToolExecutor
- SecurityPolicy

验收：

- 能完成真实 provider tool-call loop。
- 工具结果进入 MessageRuntime。
- context tools 和 external tools 路由清晰。

### Phase 4: Skills + MCP

- Skill registry
- Skill tool
- MCP registry
- MCP tool adapter

验收：

- system 只列 skill/MCP 摘要。
- 通过 tool 加载 skill。
- MCP tools 进入 capability plane。

### Phase 5: Persistence + Observability

- SQLite/File persistence
- Event log
- Langfuse adapter
- OTel adapter
- debug projection

验收：

- session 可恢复。
- message/context/compression/recall 事件可追踪。
- 默认 prompt 仍不暴露 runtime metadata。

### Phase 6: Memory + Multi-Agent + Channels

- Memory extractor/retriever/store
- subagent orchestration
- CLI/HTTP channels
- eval runner

验收：

- memory context 可召回并渲染。
- subagent 隔离运行。
- agent 可通过不同 channel 暴露。

---

## 15. 一句话架构图

```text
NeoAgent
  -> AgentLoop
      -> ContextRuntime      # agent cognition
      -> MessageRuntime      # truth source + active window
      -> CapabilityRuntime   # tools / skills / MCP
      -> ProviderRuntime     # model backend
      -> Observability       # runtime metadata, not prompt
      -> Persistence         # recovery and storage
```
