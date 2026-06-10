---
title: "Query Loop — Hermes Agent"
category: L2
parent: "[[query-loop]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的推理主循环由 `run_agent.py` 中的 `AIAgent.run_conversation()` 方法实现，以单一扁平循环驱动整个 agent turn：`while api_call_count < max_iterations and iteration_budget.remaining > 0`。相比 Claude Code 的 generator + 状态机分层设计，Hermes 将所有逻辑（API 调用、流处理、工具执行、错误恢复、上下文压缩、预算管理）压缩进同一个方法体内，形成了一个超过 2400 行的高密度单函数循环。其核心设计哲学是**防御性工程**：对每一类异常情况（无效 JSON、未知工具、截断响应、上下文溢出、速率限制、思维块签名失效）都设计了独立的检测、重试或降级路径，同时通过线程安全的 `IterationBudget` 对象让父 agent 与子 agent 共享迭代预算控制。

## 架构分析

### 主循环结构

`run_conversation()` 的执行顺序如下：

```
1. 恢复主模型运行时（上一 turn 若触发 fallback，本 turn 优先用主模型重试）
2. 清理 per-turn 状态（retry 计数器、interrupt 标志等）
3. 预检连接（清理上次 provider 异常遗留的死 TCP 连接）
4. 重置 IterationBudget（每 turn 独立预算）
5. 注入用户消息，检查 memory nudge 触发条件
6. 构建/复用 system prompt（含 session DB 缓存以保证 prefix caching 命中）
7. 预检上下文压缩（history 已超阈值时，先压缩再进循环）
8. 调用 pre_llm_call plugin hook，收集 plugin 注入的 user context
9. ── 主循环 while api_call_count < max_iterations ──
   a. 检查 interrupt 信号
   b. consume() 迭代预算（失败则退出）
   c. 触发 step_callback（gateway agent:step 事件）
   d. 构建 api_messages（注入 ephemeral context、缓存控制、消息清洗）
   e. 发起 streaming API 调用（_interruptible_streaming_api_call）
   f. 按 finish_reason 分支处理（stop / length / tool_calls）
   g. tool_calls 分支：执行工具 → 追加 tool result → 继续循环
   h. 无 tool_calls 分支：提取 final_response → break
10. 清理资源，持久化 session，触发 post_llm_call hook
11. 后台异步触发 memory/skill review（不阻塞 response 返回）
```

### 智能模型路由：按 turn 难度选模型

`agent/smart_model_routing.py` 在每次 `run_conversation()` 调用前执行，用来判断本轮是否可以绕过主模型，改用更便宜的轻量模型（cheap model）处理。整个逻辑由两个函数构成：

**`choose_cheap_model_route(user_message, routing_config)`** — 保守匹配，只在消息确定"简单"时返回 cheap model 配置：

```python
# 依次检查（任意一项命中 → 返回 None，走主模型）
if len(text) > max_chars:          return None  # 默认 160 字符
if len(text.split()) > max_words:  return None  # 默认 28 词
if text.count("\n") > 1:           return None  # 多行消息
if "```" in text or "`" in text:   return None  # 含代码块
if _URL_RE.search(text):           return None  # 含 URL
if words & _COMPLEX_KEYWORDS:      return None  # 含复杂意图词
```

`_COMPLEX_KEYWORDS` 共 40+ 词，覆盖三类意图：
- **调试/分析**：`debug`、`traceback`、`exception`、`analyze`、`investigate`、`review`
- **工程操作**：`implement`、`refactor`、`patch`、`optimize`、`plan`、`delegate`、`subagent`
- **基础设施**：`terminal`、`shell`、`tool`、`tools`、`pytest`、`docker`、`kubernetes`、`cron`

URL 检测使用正则 `https?://|www\.`（不区分大小写），覆盖 HTTP 和裸域名格式。

**`resolve_turn_route(user_message, routing_config, primary)`** — 组合函数，整合路由决策与运行时参数：若 `choose_cheap_model_route` 返回 None 则直接用主模型；否则调用 `hermes_cli.runtime_provider.resolve_runtime_provider()` 解析 cheap model 的 API key/base_url/provider，并对外返回统一的路由字典：

```python
{
    "model": "gpt-4o-mini",        # 或主模型
    "runtime": {...},              # provider/key/base_url/api_mode
    "label": "smart route → gpt-4o-mini (openai)",  # cheap 路由时有值，主模型时 None
    "signature": (model, provider, base_url, api_mode, command, args),
}
```

`signature` 字段用于检测 runtime 是否与上一 turn 相同（避免对每个 API 客户端重复构造连接）。

**配置结构（`config.yaml`）：**
```yaml
routing:
  enabled: true
  max_simple_chars: 160     # 超过则走主模型
  max_simple_words: 28
  cheap_model:
    provider: openai
    model: gpt-4o-mini
    api_key_env: OPENAI_API_KEY   # 可选，从独立环境变量读取
```

**设计取向：** routing 逻辑纯函数、无副作用，`resolve_runtime_provider` 失败时自动 fallback 到主模型（无需人工干预），`api_key_env` 支持 cheap model 用不同的账户 key，与主模型账单隔离。整套路由在 config `enabled: false`（默认）时完全透明，不影响现有行为。

### IterationBudget：线程安全迭代计数器

`IterationBudget` 是 Hermes 处理父子 agent 预算共享的核心数据结构：

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self._lock = threading.Lock()
        self._used = 0

    def consume(self) -> bool: ...   # 消耗一次迭代，返回是否允许
    def refund(self) -> None: ...    # 退还一次迭代（execute_code 免费调用）
```

- 父 agent 的 `max_iterations` 默认 90；子 agent（subagent）独立 budget，默认 50，两者**不共享**总量，而是各自独立计数
- `execute_code` 工具（程序化工具调用）执行后会调用 `refund()`，不消耗预算
- 循环入口同时检查两个条件：`api_call_count < max_iterations` AND `iteration_budget.remaining > 0`，双重保险

### 会话状态管理

`run_conversation()` 接受外部传入的 `conversation_history`，在方法内 `list()` 拷贝后作为本轮 `messages`，结束时通过 `_persist_session()` 写入 SQLite 和 JSON log。状态分为以下层次：

| 状态类型 | 持有位置 | 生命周期 |
|---------|---------|---------|
| messages（消息历史） | 方法局部变量 | 单次 run_conversation 调用 |
| _cached_system_prompt | self 属性 | 会话级（仅首 turn 构建，压缩后重建） |
| iteration_budget | self 属性 | 每 turn 重置（`IterationBudget(max_iterations)`） |
| retry 计数器 | self 属性 | 每 turn 重置（`_invalid_tool_retries = 0` 等） |
| session_*_tokens | self 属性 | 会话累计（跨 turn 持久） |
| _last_content_with_tools | self 属性 | turn 内临时（辅助 empty follow-up 恢复） |

关键设计：`_cached_system_prompt` 在 gateway 多 turn 场景下从 session DB 加载而非重建，以保证 Anthropic prefix cache 命中。

### 回调架构

Hermes 使用扁平的可选回调字段，而不是 hook 注册机制：

| 回调名 | 触发时机 | 典型用途 |
|-------|---------|---------|
| `thinking_callback(text)` | API 调用期间 | CLI TUI 显示推理进度旋转器 |
| `stream_delta_callback(delta)` | 每个流式 token | TTS 管道（提前开始语音合成） |
| `tool_start_callback(id, name, args)` | 每个工具执行前 | gateway 发送 tool.start 事件 |
| `tool_progress_callback(event, name, preview, args)` | 工具执行中/完成 | gateway 实时进度推送 |
| `tool_complete_callback(id, name, args, result)` | 每个工具执行后 | gateway 发送 tool.complete 事件 |
| `step_callback(iteration, prev_tools)` | 每次主循环迭代开始 | gateway agent:step 事件 |
| `clarify_callback(question, choices) -> str` | clarify 工具执行时 | 交互式用户澄清（CLI 或 gateway 层实现） |
| `status_callback(message)` | 状态变化时 | gateway 侧边栏状态展示 |
| `reasoning_callback` | 推理内容可用时 | 显示模型推理过程 |

所有回调调用均包裹在 `try/except Exception` 中，回调异常不会中断 agent 循环。

### 错误恢复模式

Hermes 对每类异常都有专门的检测与恢复路径：

**工具调用层面：**
- **无效工具名**：先尝试 `_repair_tool_call()` 模糊修复；失败则向模型返回 `"Tool 'x' does not exist. Available: ..."` 错误消息，让模型自我纠正；最多重试 3 次后标记为 `partial`
- **无效 JSON 参数**：3 次重试后，注入 tool error 结果（而非 user 消息），保持角色交替合法性
- **不完整的 `<REASONING_SCRATCHPAD>`**：检测到未关闭的推理标签时，不追加消息直接重试，最多 2 次

**API 层面：**
- **response 为 None 或 empty**：立即尝试切换 fallback provider，不等待 backoff
- **截断响应（`finish_reason=length`）**：先检测是否为思维预算耗尽（content 中无 `</think>` 后内容），是则直接返回用户友好错误；否则最多 3 次 continuation retry（注入 `[System: Continue where you left off]` 用户消息）
- **速率限制（429）**：解析 `Retry-After` 响应头，有则直接用；无则 jittered exponential backoff（base=2s, cap=60s）；有 fallback chain 时优先切换 provider
- **上下文溢出（413 / 400 + 大 session / 服务器断联）**：触发 `_compress_context()`，最多 3 次压缩尝试；压缩后 `api_call_count -= 1` 并 `refund()` 预算，不消耗迭代次数
- **Anthropic 思维块签名失效（400）**：清除所有消息的 `reasoning_details` 字段，单次重试
- **思维仅有内容，无可见文本**：追加 interim 消息作为"thinking prefill"继续，最多 2 次

所有等待期间以 0.2s 为粒度检查 `_interrupt_requested`，保证可中断性。

### fallback provider 链

Hermes 支持有序 fallback provider 链：

```python
self._fallback_chain = [
    {"provider": "openai", "model": "gpt-4o"},
    {"provider": "anthropic", "model": "claude-3-5-sonnet"},
]
```

`_try_activate_fallback()` 顺序尝试链中的 provider，激活后重置 `retry_count = 0`。关键设计：fallback 是 **turn-scoped** 的——每 turn 开始时 `_restore_primary_runtime()` 恢复主模型，保证优先用主模型而非固化在 fallback 上。

### 工具执行：并行与串行判定

`_execute_tool_calls()` 通过 `_should_parallelize_tool_batch()` 做并行安全判断：

**并行条件（全部满足才并行）：**
1. 工具数 > 1
2. 没有 `clarify` 工具（用户交互工具不可并行）
3. 所有工具要么在 `_PARALLEL_SAFE_TOOLS` 白名单中（`web_search`、`read_file`、`vision_analyze` 等），要么是 `_PATH_SCOPED_TOOLS`（`read_file`、`write_file`、`patch`）且目标路径不重叠

**并行执行实现：**
- `ThreadPoolExecutor(max_workers=min(num_tools, 8))`
- 结果按原始工具调用顺序收集（slots 数组 + index），追加到 messages 时保持顺序一致性
- 工具异常被 `_run_tool` 捕获，转为 error string，不崩溃线程池

**agent-level 工具（须串行）：**
`todo`、`memory`、`session_search`、`delegate_task` 在 `_invoke_tool()` 中直接处理，不经过 `model_tools.handle_function_call()`，因为它们需要访问 agent 实例状态（`_todo_store`、`_memory_store`、`_session_db`、`self`）。

### 流处理与 API 模式适配

Hermes 支持三种 API 模式，在每次迭代中统一适配：

| API 模式 | 适用场景 | 响应归一化 |
|---------|---------|---------|
| `chat_completions` | OpenAI 兼容接口（OpenRouter 等） | `response.choices[0].message` |
| `anthropic_messages` | 原生 Anthropic API | `normalize_anthropic_response()` |
| `codex_responses` | OpenAI Codex Responses API（GPT-5 等） | `_normalize_codex_response()` |

`api_mode` 在 `__init__` 中通过 `base_url` 和 `provider` 自动检测（如 `api.anthropic.com` → `anthropic_messages`，`api.openai.com` → `codex_responses`）。三种模式共享同一条主循环路径，差异由归一化层屏蔽。

流处理默认开启（即使没有 stream_delta_callback），因为流式模式支持 90 秒 stale 检测和 60 秒 read timeout，避免 provider 伪连接导致的无限挂起。

### 预算压力注入（Budget Pressure）

当迭代次数接近上限时，Hermes 将警告文本注入**工具返回结果**的 JSON body（不是新消息）：

```python
# 注入位置：最后一条 tool result 的 JSON 内容
parsed["_budget_warning"] = "BUDGET WARNING: Iteration 75/90..."
```

两个阈值：70%（caution）和 90%（urgent）。已注入的警告在下一 turn `run_conversation()` 入口通过 `_strip_budget_warnings_from_history()` 清除，防止跨 turn 泄漏影响模型行为（特别是 GPT 家族模型会把残留警告当作当前指令）。

### 上下文压缩集成

`ContextCompressor` 以 50% 上下文窗口为默认触发阈值，在两个时机检查：
1. **预检（preflight）**：`run_conversation()` 入口，检查 history 是否已超阈值
2. **内循环后**：每次工具执行完毕后，用 `last_prompt_tokens + last_completion_tokens` 实时评估

压缩后 `conversation_history = None` 重置，确保 `_flush_messages_to_session_db` 将压缩后的消息全量写入新 session，不遗漏。

## 关键代码路径

- `run_agent.py` — `AIAgent` 类，`run_conversation()` 主循环（行 6800–9200）
- `run_agent.py` — `IterationBudget` 类（行 167–209）
- `run_agent.py` — `_execute_tool_calls()` 并行/串行路由（行 5930–5951）
- `run_agent.py` — `_execute_tool_calls_concurrent()` 并行执行器（行 6027–6250）
- `run_agent.py` — `_invoke_tool()` 单工具分发（行 5953–6025）
- `run_agent.py` — `_should_parallelize_tool_batch()` 并行安全判定（行 265–306）
- `model_tools.py` — `handle_function_call()` 注册表分发 + plugin hook（行 459–548）
- `model_tools.py` — `_run_async()` sync→async 桥接（行 81–125）
- `tools/registry.py` — `ToolRegistry.dispatch()` 工具执行 + async 桥接（行 149–166）
- `agent/context_compressor.py` — `ContextCompressor` 上下文压缩逻辑
- `agent/prompt_caching.py` — `apply_anthropic_cache_control()` cache 控制注入

## 设计亮点

- **三层 async 桥接防御**：`_run_async()` 根据调用环境选择不同策略——主线程用持久 event loop，worker 线程用 thread-local 持久 loop，async 上下文用独立线程的 `asyncio.run()`。这套机制解决了 httpx/AsyncOpenAI 客户端绑定到已关闭 loop 的"Event loop is closed"错误，是工程上难得的精细设计。

- **工具结果内嵌预算警告**：把迭代压力信号注入工具返回 JSON 的 `_budget_warning` 字段，而不是注入新消息，避免破坏消息角色结构（assistant/tool/user 交替），同时跨 turn 自动清理防止信号残留。

- **并行工具执行的路径级安全判定**：`_should_parallelize_tool_batch()` 不只做工具类型白名单检查，还对 `_PATH_SCOPED_TOOLS` 做路径级重叠检测（`_paths_overlap()`），确保 `write_file` + `write_file` 写不同文件时才并行，细粒度控制优于简单的"写操作不并行"。

- **Fallback 链的 turn-scoped 设计**：fallback 激活后不永久切换，每 turn 开始时 `_restore_primary_runtime()` 总是先尝试主模型，保证"优先用偏好模型，降级是异常而非常态"的语义。

- **多 API 模式统一主循环**：chat_completions / anthropic_messages / codex_responses 三种 API 格式通过归一化层屏蔽差异，主循环代码路径完全一致，新增 provider 只需实现归一化适配层。

- **`_SafeWriter` 防崩溃 stdout**：在 systemd/Docker 等 headless 环境下，stdout pipe 可能变为不可用，任何 `print()` 调用都会抛出 `OSError`。`_SafeWriter` 透明包装 stdout/stderr，捕获 `OSError` 和 `ValueError`（线程关闭 IO），让 agent 在任何部署环境下都不因打印而崩溃。

## 局限性

- **单函数 2400 行主循环**：所有逻辑耦合在 `run_conversation()` 中，没有显式的状态转换图。`finish_reason`、`restart_*` 标志、retry 计数器等控制流散落在函数体各处，边缘 case（如压缩 + 截断 + fallback 同时触发）的行为需要阅读源码才能理解，可测试性较弱。

- **父子 agent 预算各自独立**：注释明确说明父子 budget 不共享总量，意味着一个 parent（90 次）派发 5 个 subagent（各 50 次）时，实际总迭代可高达 340 次，远超用户预期的 90 次上限。

- **工具执行结果直接 append 无大小控制**：工具返回 JSON string 直接追加到 messages，没有类似 Claude Code `applyToolResultBudget()` 的体积控制层。虽然有 `enforce_turn_budget()` 做 per-turn 聚合预算，但单个过大的工具返回仍可能直接膨胀上下文。

- **串行工具对 clarify 的特殊处理**：`_NEVER_PARALLEL_TOOLS = frozenset({"clarify"})` 强制整批工具退回串行，而不是只对 `clarify` 本身串行。如果模型在同一 turn 同时调用 `clarify` 和多个 `read_file`，后者也被迫串行执行。

- **工具调用 schema 动态注入的副作用**：`get_tool_definitions()` 在 `__init__` 时静态生成，memory provider plugin 的 schema 是后期追加到 `self.tools` 的，但 `_last_resolved_tool_names` 是进程全局变量，多个并发 AIAgent 实例（如 gateway 多会话）可能相互干扰该全局状态。

## 来源

- 源码版本：hermes-agent 0.16.0 (pyproject.toml)
- 核心文件：`run_agent.py`、`model_tools.py`、`tools/registry.py`
- 分析深度：源码级
