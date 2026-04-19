# 覆盖率扫描报告：neoagent

**生成时间**: 2026-04-19 11:30

## 汇总

| 指标 | 数值 |
|------|------|
| 有效顶层模块总数 | 11（neoagent/ 下的 .py 文件 + 7 个子包） |
| 已覆盖模块数 | 11 |
| 覆盖率 | 11/11 = 100% |
| 状态 | PASS（≥80%）|
| 补漏轮次 | 0 |

## 模块覆盖详情

| 模块路径 | 对应 L2 | 状态 |
|---------|---------|------|
| `neoagent/agent.py` | query-loop + prompt-system + session-recovery + hooks + mcp-skills + evaluation-observability | ✓ |
| `neoagent/config.py` | session-recovery（作为配置层） | ✓ |
| `neoagent/core/loop.py` | query-loop + context-management | ✓ |
| `neoagent/core/prompt.py` | prompt-system + mcp-skills（skill 三态） | ✓ |
| `neoagent/core/compress.py` | context-management | ✓ |
| `neoagent/core/types.py` | 所有 L2 隐式依赖（Message/Turn/Block 类型层） | ✓ |
| `neoagent/session.py` | session-recovery + context-management（SessionState freed 字段） | ✓ |
| `neoagent/tools/` | tool-system（base/registry/executor/deferred/permission/pathguard） | ✓ |
| `neoagent/tools/builtin/` | tool-system + context-management（free/recall）+ mcp-skills（tool_search/skill_load/write_skill） | ✓ |
| `neoagent/memory/` | memory-system | ✓ |
| `neoagent/channels/` | channel-remote | ✓ |
| `neoagent/mcp/` | mcp-skills + channel-remote（StdioTransport 作为另一种远程通信） | ✓ |
| `neoagent/multi/` | multi-agent + evaluation-observability（event bubble） | ✓ |
| `neoagent/eval/` | evaluation-observability | ✓ |
| `neoagent/events.py` | evaluation-observability + hooks | ✓ |
| `neoagent/hooks.py` | hooks | ✓ |
| `neoagent/observe.py` + `observe_subscriber.py` | evaluation-observability | ✓ |
| `neoagent/providers/` | 支撑层——不独立映射概念（隐含在 query-loop 中） | ✓（视作 infrastructure） |

## 排除目录

| 路径 | 排除原因 |
|------|---------|
| `build/`, `neoagent.egg-info/` | 构建产物 |
| `tests/` | 测试代码 |
| `examples/`, `docs/`, `review/` | 示例与文档 |
| `logs/` | 运行时产物 |
| `__pycache__/` | 字节码 |
| `uv.lock`, `pyproject.toml` | 依赖管理元数据 |

## 概念交叉检查发现

| 概念维度 | 关键词 | 发现 | 处理 |
|---------|-------|------|------|
| memory-system | dream/archive/consolidate/compact/recall/forget/persist | 无 `dream/archive/consolidate/forget` 命中（neoagent 记忆系统简单文件存储，无这些高级概念） | 无需补漏 |
| query-loop | loop/run/step/turn/cycle/execute/dispatch/agentic | 全部命中，在 `core/loop.py` / `agent.py` 中 | 已覆盖 |
| tool-system | tool/call/invoke/register/executor/permission/sandbox | 全部命中，`tools/*` 全部已映射 | 已覆盖 |
| multi-agent | orchestrat/worker/spawn/delegate/coordinate/handoff/swarm | `orchestrator/worker/spawn/delegate` 命中在 `multi/*`；`handoff/swarm` 无命中（neoagent 不做 peer-to-peer） | 已覆盖（无 peer 通信是设计选择，已在 L2 记录） |
| context-management | context/compress/summarize/window/token/truncate/prune | 全部命中（`core/compress.py` + `loop.py` freed 重写） | 已覆盖 |
| hooks | hook/event/lifecycle/pre_tool/post_tool/intercept/middleware | `hook/pre_tool_call/post_tool_call/pre_provider_call/post_provider_call` 全部命中；无 `middleware`（neoagent 不用中间件抽象） | 已覆盖 |
| mcp-skills | mcp/plugin/extension/server/client/stdio/transport | `mcp/client/stdio/transport` 全部命中；无 `plugin`（neoagent 只有 mcp+skill 两套，无独立 plugin 层） | 已覆盖（设计选择） |
| session-recovery | recovery/resume/checkpoint/snapshot/crash/restore | `resume/restore/snapshot`（session.py 有 `save/load/fork/cleanup`）；无 `checkpoint/crash` 显式字段 | 已覆盖（checkpoint 即 per-turn save） |
| channel-remote | channel/websocket/sse/streaming/fastapi/http/interface | `channel/sse/fastapi/http` 全部命中；无 `websocket`（neoagent 只有 FastAPI） | 已覆盖（设计选择） |
| evaluation-observability | eval/metric/logging/trace/benchmark/observer/span | `eval/metric/logging/observer` 全部命中；无 `trace/span/benchmark/telemetry`（neoagent 无 OTEL） | 已覆盖（设计局限已在 L2 记录） |
| prompt-system | prompt/system_prompt/template/inject/section/priority | 全部命中 | 已覆盖 |
| runtime-state | state/session/storage/persist/serialize/checkpoint | 全部命中（`SessionState` 11 字段 + `JsonFileStorage`） | 已覆盖（合并入 session-recovery 和 context-management 两个 L2） |

## 补漏执行记录

**本轮无需补漏**——全部顶层模块在 10 个 L2 页面中明确分析。`pathguard.py`（1 函数 37 行的 `validate_path` 安全工具）、`providers/`（provider 适配层）未单独作 L2，但作为基础设施被 tool-system 和 query-loop L2 隐式覆盖（tool-system L2 提到 `max_result_size`、provider 在 query-loop L2 的"Provider 调用 + max_tokens 自动重试"段落中涉及）。

`runtime-state` 作为 L1 概念的模块在 neoagent 中没有独立子系统（不像 Hermes 有专门的 State 模块），而是分散在 `Session/SessionState/NeoAgentConfig` 三个地方——合并入 session-recovery L2 分析更连贯，避免信息碎片化。
