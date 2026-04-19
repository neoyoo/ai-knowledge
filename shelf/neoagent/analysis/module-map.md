# neoagent 模块映射

**生成时间**: 2026-04-19
**源路径**: /Users/neo/Desktop/project/git/neoagent/
**主语言**: Python (async)
**代码规模**: ~6200 LoC (`neoagent/` 包)
**git 版本**: 9d3621d (2026-04-13)

## 顶层模块清单

| 模块路径 | 主要功能（推断） | 映射概念维度 |
|---------|------------------|-------------|
| `neoagent/agent.py` | `NeoAgent` 主入口，组合全部子系统（407 行） | query-loop, prompt-system, hooks, mcp-skills, session-recovery, evaluation-observability |
| `neoagent/config.py` | `NeoAgentConfig` dataclass（memory_dir / session_dir / provider / system_prompt 等） | runtime-state |
| `neoagent/core/loop.py` | `QueryLoop.run()` 主循环：压缩检查、provider 调用、tool 执行、freed tool result 重写、per-turn auto-save、auto_free_after 衰老追踪 | query-loop, context-management |
| `neoagent/core/prompt.py` | `PromptBuilder` + `PromptSection`：优先级排序、register/activate/deactivate skill 三态流转 | prompt-system, mcp-skills |
| `neoagent/core/compress.py` | `ContextCompressor`：tiktoken 计数、70% 预算阈值、LLM 摘要 + 前次摘要延续、失败 3 次降级截断、`_sanitize_tool_pairs()` 保证 tool 配对 | context-management |
| `neoagent/core/types.py` | `Message`/`TextBlock`/`ToolUseBlock`/`ToolResultBlock`/`Turn`/`ConversationResult` Pydantic 类型层 | runtime-state |
| `neoagent/session.py` | `Session`/`SessionState`/`FreedToolResult`/`JsonFileStorage`（含 `cleanup` 清理）；save_if_storage per-turn 自动持久化 | session-recovery, runtime-state, context-management |
| `neoagent/tools/base.py` | `BaseTool`：`auto_free_after: int = 0` 字段 + `preview()` 方法 + `permission`（auto/ask/deny）+ `is_concurrent_safe` | tool-system, context-management |
| `neoagent/tools/registry.py` | `ToolRegistry`：纯数据结构 register/get_tool/get_schemas（deny 不暴露）/unregister | tool-system |
| `neoagent/tools/executor.py` | `ToolExecutor`：并发分区（safe 走 asyncio.gather，unsafe 串行）+ PermissionChecker + pre/post_tool_call hook + 截断 + ToolCall/ResultEvent 发射 | tool-system, hooks |
| `neoagent/tools/deferred.py` | `DeferredToolRegistry`：register/promote/is_deferred/search 三种查询模式（select:/+kw/regex）；MCP 工具隐藏机制的核心 | tool-system, mcp-skills |
| `neoagent/tools/permission.py` | 三级权限检查器（auto/ask/deny + auto_approve fallback） | tool-system |
| `neoagent/tools/pathguard.py` | （未读取，推测为路径白名单防护） | tool-system |
| `neoagent/tools/builtin/` | 内置工具集：bash/read/edit/write/glob/grep + run_python（sandbox）+ skill_load + write_skill + tool_search + free_tool_result + recall_tool_result | tool-system, context-management, mcp-skills |
| `neoagent/memory/` | extractor（5 tool calls 或 4K token delta 触发 LLM 提取）+ retriever（lexical 评分）+ store（file-based，MEMORY.md 索引）+ manager | memory-system |
| `neoagent/channels/` | `Channel` ABC + `FastAPIChannel`（POST /v1/run 同步 + SSE streaming + Bearer auth + CORS + asyncio.Queue 同步 bus→async gen 桥接） | channel-remote |
| `neoagent/mcp/` | `MCPClient`（JSON-RPC 2.0 + `_pending` 乱序响应缓冲）+ `MCPTool`（`_schema_to_pydantic` 递归 JSON Schema 转换，支持 array/object/enum）+ `StdioTransport`（subprocess + stderr drain + env 白名单） | mcp-skills, channel-remote |
| `neoagent/multi/` | `Orchestrator`（brain = NeoAgent + 5 个内部工具 + semaphore 并发上限）+ `WorkerCard` frozen dataclass + `Task/TaskResult/TaskTracker`（asyncio.timeout + Cancelled 处理 + work_summary 提取）+ `spawn_worker`/`delegate_task`/`cancel_task` 内部工具 + event bubble | multi-agent |
| `neoagent/multi/events.py` | `_setup_event_bubble`：worker event bus 的 `subscribe_all` 包成 `WorkerEvent(worker_name, task_id, depth, inner)` 发射到 orchestrator bus + teardown 返回 unsubscribe 函数 | multi-agent, evaluation-observability |
| `neoagent/eval/metrics.py` | `MetricsCollector` + `TurnMetrics`/`SessionMetrics`：订阅四事件积累 turn 数据、latency_ms 取 request_time 到 response_time | evaluation-observability |
| `neoagent/eval/runner.py` | `EvalRunner/EvalCase/EvalCaseResult/EvalReport`：每 case 独立 `MetricsCollector`，try/finally 保证 unsubscribe，异常不中断批次 | evaluation-observability |
| `neoagent/eval/usage.py` | `UsageTracker`：per-model token 累加 | evaluation-observability |
| `neoagent/events.py` | `EventBus`（dict 订阅 + handler 异常不中断）+ 16 种 frozen dataclass 事件类型（Provider/Tool/Compress/Memory/Skill/Turn/Worker/Task/SessionResume/ToolResult 生命周期） | evaluation-observability, hooks, multi-agent |
| `neoagent/hooks.py` | `HookManager` + `HookResult(allow/deny/modify)` + 4 种 hook 事件（pre/post × tool/provider）+ MappingProxyType 强制不可变 + dataclasses.replace 实现 modify | hooks |
| `neoagent/observe.py` | `Observer`：角色着色（SYSTEM/USER/ASSISTANT/TOOL）+ 文件日志 + stderr/console 双写 + 压缩/记忆/freed/skill 专用行 | evaluation-observability |
| `neoagent/observe_subscriber.py` | `ObserverSubscriber`：EventBus → Observer 方法的纯适配器，attach/detach 成对 | evaluation-observability |
| `neoagent/providers/` | `Provider` ABC + `AnthropicProvider` + `OpenAIProvider`（处理 tool_choice、content block 转换） | （不独立映射，作为支撑层） |

## 概念维度覆盖计划

| 概念维度 | 相关模块 | 是否产出 L2 |
|---------|---------|------------|
| query-loop | `core/loop.py`, `agent.py` | 是 |
| prompt-system | `core/prompt.py`, `agent.py::enable_memory/system_prompt` | 是 |
| tool-system | `tools/*`, `tools/builtin/*` | 是 |
| context-management | `core/compress.py`, `tools/builtin/free_tool_result.py`, `tools/builtin/recall_tool_result.py`, `core/loop.py`（freed 重写段） | 是 |
| memory-system | `memory/*` | 是 |
| session-recovery | `session.py`, `agent.py::resume/_validate_resume` | 是 |
| hooks | `hooks.py`, `events.py`（EventBus）, `tools/executor.py` hook 穿插 | 是 |
| channel-remote | `channels/*`, `mcp/transport.py` | 是 |
| mcp-skills | `mcp/*`, `tools/deferred.py`, `tools/builtin/tool_search.py`, `tools/builtin/skill_load.py`, `tools/builtin/write_skill.py`, `core/prompt.py::register/activate_skill` | 是 |
| multi-agent | `multi/*` | 是 |
| evaluation-observability | `events.py`, `observe.py`, `observe_subscriber.py`, `eval/*`, `multi/events.py` | 是 |
| runtime-state | `session.py::SessionState`, `tools/builtin/tool_search.py::ContextVar`, `config.py` | 合并入 session-recovery 与 tool-system（不单独写 L2） |
| finetuning-system | 无 | 否（neoagent 无微调模块） |

## 排除目录

| 路径 | 排除原因 |
|------|---------|
| `build/`, `neoagent.egg-info/` | 构建产物 |
| `tests/` | 测试代码 |
| `examples/`, `docs/`, `review/` | 示例与文档 |
| `logs/` | 运行时产物 |
| `__pycache__/` | 字节码 |
| `uv.lock`, `pyproject.toml` | 依赖管理元数据 |
| `neoagent/providers/` | 作为 provider 适配层，不独立映射概念（在 query-loop/tool-system 中被动涉及） |
| `neoagent/__init__.py`, `neoagent/multi/__init__.py` | 空 re-export |
