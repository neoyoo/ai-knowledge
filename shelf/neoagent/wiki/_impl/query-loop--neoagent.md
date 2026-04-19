---
title: "query-loop — neoagent"
category: L2
parent: "[[query-loop]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: query-loop
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的 `QueryLoop.run()` 是一个典型的 Python async 单循环（`core/loop.py`，340 行），但融合了 7 类横切关注点：context 压缩触发、deferred MCP 工具过滤、freed tool-result 重写、pre/post provider hook、memory 抽取、per-turn auto-save、auto_free_after 衰老追踪。循环以 Session 为一等公民（而非简单 `list[Message]`），通过 `SessionState` 跨轮持久化所有状态；stop_reason == "end_turn" 或无 tool_use 即退出，否则串 `ToolExecutor.execute()` 后进入下一轮。独特点：**freed/recall 机制把上下文管理从"一次性压缩"演进为"细粒度 tool_result 生命周期管理"**，是这套循环最有信息量的设计。

## 架构分析

### 单循环职责分层

`QueryLoop.__init__` 注入 provider、tool_registry、tool_executor、prompt_builder、memory_manager、event_bus、hook_manager、deferred_registry 八个依赖。`context_budget` 为 0 时回退到 `provider.get_context_window()`，这是唯一的自动配置逻辑。循环体每一轮按序执行：

1. **压缩检查**：`ContextCompressor.should_compress(msgs, schemas, budget)` → 触发则调用 `compress()`（LLM 摘要，失败时 `_truncate_oldest`），压缩次数由 `session_state.compression_failures` 累计，达到 `max_failures=3` 后永久走截断路径。
2. **Prompt 构建 + deferred 注入**：`PromptBuilder.build()` 得到基础 system prompt，然后如果存在 `deferred_registry`，循环体计算每个工具的隐藏状态（session-scoped `promoted_tools` OR 全局 `is_deferred()`），过滤 schemas，并把仍 deferred 的工具名以 `<deferred-tools>...</deferred-tools>` 段附加到 system prompt 末尾。
3. **Freed tool result 重写**：如果 `session_state.freed_tool_results` 非空，调用 `_apply_freed_to_messages(msgs, freed, recalled_this_turn)` 得到 `msgs_for_llm`（原 msgs 不被修改，只是 provider 收到的副本中 freed 且未 recall 的 `ToolResultBlock` 被换成 placeholder）；同时把 `_render_freed_section(freed)` 作为 "## Freed Tool Results (recoverable)" 追加到 system prompt。
4. **pre_provider_call hook**：`HookManager.run_pre("pre_provider_call", PreProviderCallEvent)` 返回 deny 时直接生成 "[Hook denied: reason]" assistant 消息退出；返回 modify 则应用到 system/messages/tools 副本。
5. **Provider 调用 + max_tokens 自动重试**：如果 stop_reason == "max_tokens"，用 `_retry_with_higher_max`（`_DEFAULT_MAX_TOKENS * 2`）再试一次；还是截断则作为 end_turn 退出（避免 corrupt tool_calls）。
6. **post_provider_call hook**：可替换 response。
7. **分支**：无 tool_calls 或 stop_reason == "end_turn" → `memory_manager.maybe_extract()` 检查是否触发抽取，emit MemoryExtractEvent，append assistant message，调 `_session.save_if_storage()`，返回 ConversationResult。
8. **tool_use 分支**：`ToolExecutor.execute(tool_calls)` → append assistant message + user message（ToolResultBlock 数组）→ 记录 `tool_use_id → tool_name` 映射到 `session_state.tool_use_to_tool_name`。
9. **Auto_free_after 衰老**：对所有非 freed 的 `ToolResultBlock` 的 `tool_use_id`，`session_state.tool_result_ages[id] += 1`；然后扫描所有 age，若对应 tool 的 `auto_free_after > 0` 且 age >= 阈值，复制 `original_content` 到 `FreedToolResult(id, tool_name, size, preview, original_content)` 写入 `session_state.freed_tool_results`，emit `ToolResultFreedEvent(reason="auto_free_after")`。
10. **Per-turn 持久化**：`_session.save_if_storage()` 无条件写 JSON（如果 `bind_storage` 过）。

### ContextVar 驱动的 session 传播

`QueryLoop.run()` 开头通过 `set_session_state(session_state)` 和 `set_current_session(_session)` 写入两个模块级 `ContextVar`（`tools/builtin/tool_search.py`），这样 `ToolSearchTool.execute()`、`FreeToolResultTool.execute()`、`RecallToolResultTool.execute()` 可以无入参访问当前 session——对 FastAPI 并发请求场景，每个 asyncio task 各自继承独立的 ContextVar 快照，不会互相覆盖。

### 关键代码路径

- `neoagent/core/loop.py:50-340` — `QueryLoop.run()` 主循环
- `neoagent/core/loop.py:347-384` — `_apply_freed_to_messages()`：返回 messages 的副本，把 freed 且未 recall 的 `ToolResultBlock.content` 替换为 `f"[freed: tool_use_id={id}, tool={tool_name}, size=NNN B, preview=...]"`
- `neoagent/core/loop.py:387-397` — `_render_freed_section()`：生成 "## Freed Tool Results (recoverable)" + 逐条 `- \`{id}\` · {tool_name} · {size}B · {preview!r}` + 尾部使用说明
- `neoagent/core/loop.py:296-337` — auto_free_after 衰老循环
- `neoagent/agent.py:128-177` — `NeoAgent.chat()` / `run()`：创建或接管 Session、绑定 storage（仅在非 temp session 时）、调用 `_loop.run(session=session)`
- `neoagent/agent.py:85-126` — `new_session()` / `resume(session_id, validate)` / `_validate_resume()`：resume 校验 workspace_path 是否存在 + 是否 stale（>24h），发射 `SessionResumeWarningEvent` 但不抛错

## 设计亮点

- **Turn-based 显式 auto_free_after 衰老**：`tool_result_ages` 字典按 tool_use_id 计数，每轮 tool_use 分支后 +1，超过该工具的 `auto_free_after` 后自动转为 freed placeholder。不同于"达到 context 阈值才压缩"的被动策略，这是一种**主动按时间衰减的工具结果生命周期管理**，让 run_python（`auto_free_after=2`）、recall_tool_result（`auto_free_after=1`）、普通 MCP 调用（默认 0 = 永不）有区别对待。
- **循环内不 mutate 消息、只生成 provider 视图副本**：`_apply_freed_to_messages` 返回新列表，`msgs` 本身始终保留原始 ToolResultBlock 内容——这样 `recall_tool_result` 可以从原 messages 直接拉回原文，而不是从某个额外存储读取；同时 session 持久化的是原文而非 placeholder，重启后 state 完整。
- **max_tokens 自动重试 + 最终降级为 end_turn**：避免模型在 tool_use 时被截断产出半截 `ToolUseBlock` 导致下一轮 tool_result 配对失败。
- **ContextVar 代替 instance 属性**：`tool_search.py` 注释中明写"replaces instance attribute"——这是 FastAPI 并发安全的关键修复，说明团队踩过并发请求状态污染的坑。
- **Deferred tool 过滤同时走 session-scoped + 全局 OR 逻辑**：`session.promoted_tools` 是主路径，全局 `is_deferred()` 保留作 backward-compat；注释里显式说"visible when EITHER condition indicates promoted"。

## 局限性

- **max_turns 默认 30，无自适应**：和 ClaudeCode 的固定上限一样，没有根据任务复杂度动态调整。
- **单循环体已近 300 行**：压缩、deferred 过滤、freed 重写、hook、memory、auto_free 全压在 `QueryLoop.run()` 内部，按 Hermes Agent 路线演化——清晰可测试但扩展新切面需要修改主循环本身。循环内各段之间的顺序隐式硬编码（如 auto_free 必须发生在 tool_result append 之后），新增切面会面临"插在哪"的设计决策。
- **没有中断点机制**：与 AgentScope 的 `CancelledError + __call__/reply 分离`不同，neoagent `run()` 内部没有显式 checkpoint；外部只能通过 asyncio task cancel 整体取消，中断后无 `handle_interrupt` 钩子生成语义化响应。
- **压缩失败后无熔断恢复**：compression_failures 达到 3 后永久走 `_truncate_oldest`，没有"冷却期过后重试 LLM 压缩"逻辑；对长 session 可能永久降级。
- **tool_result_ages 只计永久存活的 id**：当 compressor 把旧 tool_use/tool_result 对整体 sanitize 掉后，对应的 age 键会成为无意义残留（无 cleanup 逻辑），长 session 存在字典无界增长风险。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db（2026-04-13）
- 分析深度：源码级
- 关键文件：`neoagent/core/loop.py`, `neoagent/agent.py`, `neoagent/session.py`
