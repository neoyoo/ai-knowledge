---
title: "Multi-Agent — Hermes Agent"
category: L2
parent: "[[multi-agent]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的多智能体系统以 `delegate_task` 工具为核心入口，支持单任务委托和批量并行委托两种模式。与 Claude Code 的正式任务系统不同，Hermes 的委托是同步阻塞的：父 agent 调用 `delegate_task` 后，主线程阻塞直到所有子 agent 完成，然后只把子 agent 的摘要（不是中间工具调用历史）写回父 agent 上下文。设计哲学是"隔离上下文、限制能力、共享凭证池"。

此外，Hermes 还实现了独立的 `mixture_of_agents_tool`（MoA），用不同 frontier 模型并行生成答案后再由聚合模型合并，解决单模型能力天花板的问题，与 delegate_task 是互补的两条多模型协作路径。

## 架构分析

### delegate_task — 委托入口与双模式

`tools/delegate_tool.py` 中的 `delegate_task()` 函数是多智能体的统一入口。

**两种调用模式：**
- **单任务模式**：传入 `goal`（+ 可选 `context`、`toolsets`），在主线程直接运行，无线程池开销。
- **批量并行模式**：传入 `tasks` 数组（最多 `MAX_CONCURRENT_CHILDREN = 3` 项），用 `ThreadPoolExecutor(max_workers=3)` 并发执行，通过 `as_completed()` 流式收集结果。

```python
MAX_CONCURRENT_CHILDREN = 3
MAX_DEPTH = 2  # parent(0) → child(1) → reject(2)
DEFAULT_MAX_ITERATIONS = 50
```

深度限制在函数入口立即检查：
```python
depth = getattr(parent_agent, '_delegate_depth', 0)
if depth >= MAX_DEPTH:
    return json.dumps({"error": "Delegation depth limit reached ..."})
```

### _build_child_agent — 子 Agent 构造

`_build_child_agent()` 在主线程（非子线程）完成所有子 agent 的构造，然后再交给线程池执行。这个设计避免了多线程并发修改进程级全局变量 `model_tools._last_resolved_tool_names` 的竞态问题。

**工具集继承与隔离：**
子 agent 的工具集是父 agent 工具集的子集，取两者的交集，然后强制剥离被禁止的工具集：

```python
DELEGATE_BLOCKED_TOOLS = frozenset([
    "delegate_task",   # 禁止递归委托
    "clarify",         # 禁止用户交互
    "memory",          # 禁止写入共享 MEMORY.md
    "send_message",    # 禁止跨平台副作用
    "execute_code",    # 子 agent 应逐步推理，不写脚本
])
```

`_strip_blocked_tools()` 在工具集名称层面过滤（`"delegation"`, `"clarify"`, `"memory"`, `"code_execution"`），配合 `DELEGATE_BLOCKED_TOOLS` 在函数名层面双重拦截。

**关键构造参数（`AIAgent.__init__`）：**
```python
child = AIAgent(
    ephemeral_system_prompt=child_prompt,  # 子 agent 专用 system prompt
    quiet_mode=True,                        # 不打印进度到终端
    skip_context_files=True,                # 不注入 SOUL.md / AGENTS.md
    skip_memory=True,                       # 不加载/写入记忆
    clarify_callback=None,                  # 禁用用户交互
    max_iterations=effective_max_iter,      # 独立 iteration budget（默认 50）
    iteration_budget=None,                  # fresh budget，不共享父 budget
    parent_session_id=parent_agent.session_id,  # 关联父会话 ID（SQLite 记录）
    ...
)
child._delegate_depth = parent_agent._delegate_depth + 1
```

`ephemeral_system_prompt` 是"临时 system prompt"——在执行中生效，但不写入轨迹文件（trajectory），防止子 agent 的内部指令污染训练数据。

### 每次对话的 task_id 隔离

`run_conversation()` 的签名包含可选 `task_id` 参数。每次调用若未提供则自动生成：
```python
effective_task_id = task_id or str(uuid.uuid4())
```

`task_id` 用于隔离子 agent 专属的终端会话（VM）和浏览器实例，对话结束后通过 `_cleanup_task_resources(task_id)` 统一清理。父 agent 的 `task_id` 与子 agent 不同，各自独立。

### IterationBudget — 独立预算，非共享

`run_agent.py` 中的 `IterationBudget` 是线程安全的计数器：

```python
class IterationBudget:
    def __init__(self, max_total: int):
        self.max_total = max_total
        self._used = 0
        self._lock = threading.Lock()

    def consume(self) -> bool:  # 消耗一次，返回是否允许
    def refund(self) -> None:   # 退还（execute_code 调用退款）
```

注释明确说明：**每个子 agent 有独立的 budget，不从父 agent 扣减**。父 agent 默认 90 次，子 agent 默认 50 次，总计可超过 90 次。用户通过 `config.yaml` 的 `delegation.max_iterations` 控制子 agent 上限。

预算压力注入（两档阈值）：
- 70% 进度：`CAUTION` 提示，建议开始收尾
- 90% 进度：`WARNING` 提示，强制立即返回最终答案

警告文本注入到最后一条 `tool` 消息的 JSON 内容中（`_budget_warning` 字段），而不是作为独立消息，以避免破坏消息结构和 prompt caching。

### 进度回调 — 子 agent 工具调用的实时透传

`_build_child_progress_callback()` 构造一个回调，将子 agent 的工具调用事件透传给父 agent 的显示层：

- **CLI 路径**：向父 agent 的 `_delegate_spinner` 写入树状视图行（`├─ 🔧 tool_name "preview"`）
- **Gateway 路径**：批量缓冲工具名（每 5 个刷新一次），通过父 agent 的 `tool_progress_callback` 转发
- **思考事件**：`_thinking` / `reasoning.available` 事件显示截断后的推理摘要（≤55 字符）

这是一个"可观测性透传"机制，父 agent 上下文不包含子 agent 历史，但用户界面可以实时看到子 agent 在做什么。

### 结果聚合 — 父 agent 只看摘要

`_run_single_child()` 调用 `child.run_conversation(user_message=goal)`，等待完成后构建结构化结果：

```python
entry = {
    "task_index": task_index,
    "status": "completed" | "failed" | "interrupted",
    "summary": result.get("final_response"),  # 子 agent 的最终回复
    "api_calls": ...,
    "duration_seconds": ...,
    "exit_reason": "completed" | "max_iterations" | "interrupted",
    "tokens": {"input": ..., "output": ...},
    "tool_trace": [...],  # 工具调用名称 + 字节数，无完整内容
}
```

`tool_trace` 只记录工具名和参数字节数，不含完整工具调用内容。父 agent 拿到的是 `summary`（子 agent 的自然语言最终回复），不是子 agent 的完整消息历史。

结果以 JSON 字符串返回给父 agent：
```python
return json.dumps({"results": results, "total_duration_seconds": total_duration})
```

### 中断传播

`AIAgent.interrupt()` 方法在传播中断时会级联到所有正在运行的子 agent：

```python
def interrupt(self, message=None):
    _set_interrupt(True)
    with self._active_children_lock:
        children_copy = list(self._active_children)
    for child in children_copy:
        child.interrupt(message)
```

子 agent 在 `_build_child_agent()` 构造完成后立即注册到父 agent 的 `_active_children` 列表，在 `_run_single_child()` 的 `finally` 块中注销。

### 凭证池 — 多子 agent 共享轮换

父子 agent 共享 `_credential_pool`（`agent/credential_pool.py`）：同 provider 时子 agent 继承父 agent 的 pool（共享冷却状态和轮换逻辑）；不同 provider 时加载该 provider 专属的 pool。

每个子 agent 在执行前 `acquire_lease()`，执行后 `release_lease()`，使多子 agent 并发时能跨 API key 轮换，避免单 key 频率限制。

### ACP 传输层 — 跨 agent 异构委托

`delegate_task` 支持 `acp_command` / `acp_args` 参数，可让非 ACP 父 agent 启动 ACP 子 agent（如 `claude`、`copilot`）。子 agent 的构造参数直接透传：

```python
child = AIAgent(
    acp_command=effective_acp_command,
    acp_args=effective_acp_args,
    ...
)
```

AIAgent 初始化时检测 `provider == "copilot-acp"` 并设置对应的 `client_kwargs["command"]`，切换到 ACP JSON-RPC 传输。这意味着 Hermes 可以从 CLI / Discord / Telegram 父 agent 向任意 ACP 兼容 agent 委托任务。

### Mixture-of-Agents — 独立的多模型合成路径

`tools/mixture_of_agents_tool.py` 实现了基于 arXiv:2406.04692 论文的 MoA 架构：

**固定两层架构：**
1. **Layer 1（参考层）**：4 个参考模型并行生成答案（`asyncio.gather`），温度 0.6
2. **Layer 2（聚合层）**：1 个聚合模型合成最终答案，温度 0.4

```python
REFERENCE_MODELS = [
    "anthropic/claude-opus-4.6",
    "google/gemini-3-pro-preview",
    "openai/gpt-5.4-pro",
    "deepseek/deepseek-v3.2",
]
AGGREGATOR_MODEL = "anthropic/claude-opus-4.6"
```

所有调用都走 OpenRouter（需要 `OPENROUTER_API_KEY`），参考模型启用 `reasoning: {enabled: true, effort: "xhigh"}`。每个参考模型最多重试 6 次，指数退避（2s → 4s → 8s → ... → 60s）。只要有 `MIN_SUCCESSFUL_REFERENCES = 1` 个模型成功就继续聚合，避免单点失败阻塞整个流程。

MoA 与 `delegate_task` 的根本区别：
- `delegate_task`：生成新 AIAgent 实例，每个子 agent 可调用工具，适合需要主动执行操作的任务
- `mixture_of_agents_tool`：纯 LLM 推理合成，无工具调用，适合纯推理/答案合成场景

## 关键代码路径

- `tools/delegate_tool.py` — 委托入口、子 agent 构造、并行执行、结果聚合、进度回调
  - `delegate_task()` — 主函数，深度检查、模式分发、ThreadPoolExecutor
  - `_build_child_agent()` — 子 AIAgent 实例化，工具集过滤，凭证解析
  - `_run_single_child()` — 单子 agent 运行，结果结构化，工具 trace 构建
  - `_build_child_system_prompt()` — 子 agent system prompt，含 workspace 路径提示
  - `_build_child_progress_callback()` — CLI/Gateway 进度透传
  - `_resolve_child_credential_pool()` — 凭证池继承/加载
  - `DELEGATE_BLOCKED_TOOLS` — 禁止工具集合（frozenset）
- `run_agent.py`
  - `AIAgent.__init__()` — `_delegate_depth`、`_active_children`、`IterationBudget` 初始化
  - `AIAgent.interrupt()` — 级联中断传播
  - `AIAgent.run_conversation()` — `task_id` 生成，`iteration_budget` 重置
  - `IterationBudget` — 线程安全独立预算计数器
  - `_get_budget_warning()` — 70%/90% 两档压力注入
- `tools/mixture_of_agents_tool.py`
  - `mixture_of_agents_tool()` — 主函数，asyncio 并行，两层聚合
  - `_run_reference_model_safe()` — 单参考模型调用，指数退避重试
  - `_run_aggregator_model()` — 聚合模型调用，空内容重试

## 设计亮点

**1. 子 agent 工具集严格受限于父 agent**

`_build_child_agent()` 将 `toolsets` 参数与父 agent 的 `enabled_toolsets` 取交集，子 agent 不可能获得父 agent 没有的工具。这一规则在代码层面强制实施，而不是依赖 system prompt 的软约束。

**2. 主线程串行构造、子线程并行执行**

所有子 agent 在主线程构造完毕后再提交给线程池，消除了并发修改 `model_tools._last_resolved_tool_names` 全局变量的竞态。`finally` 块确保即使构造中途失败，全局变量也能恢复到父 agent 的工具名列表：

```python
finally:
    _model_tools._last_resolved_tool_names = _parent_tool_names
```

**3. ephemeral system prompt 不污染轨迹**

子 agent 的 system prompt 通过 `ephemeral_system_prompt` 注入，而不是写入对话消息列表，因此不会出现在保存的 trajectory 文件中。这一设计在训练数据质量和执行时信息安全之间取得了平衡。

**4. 中断级联到所有深度**

`interrupt()` 是递归的：父 agent 被中断时，会遍历 `_active_children` 列表，对每个运行中的子 agent 调用 `child.interrupt()`。子 agent 本身若有孙子 agent（虽然当前深度限制会阻止，但结构上支持），也会继续传播。

**5. MoA 与 delegate_task 互补**

两条多模型路径在使用场景上界限清晰：MoA 专注纯推理合成（无副作用），delegate_task 专注需要工具执行的实际操作。LLM 在 `DELEGATE_TASK_SCHEMA` 的描述中可读到这两条路径的区别，便于在调用时自主选择。

## 局限性

**1. 父 agent 同步阻塞**

委托是同步阻塞的：父 agent 在 `delegate_task` 调用期间完全挂起，无法处理用户输入或并发任务。批量模式使用 `ThreadPoolExecutor` 并发子 agent，但父 agent 的主 agent loop 本身不继续前进。这与 Claude Code 的 task 系统（后台异步、可 resume）形成对比。

**2. 固定深度上限 2，不可配置**

`MAX_DEPTH = 2` 硬编码，孙子 agent 被直接拒绝。对于需要三级以上层级分工的复杂任务，只能通过 batch 模式在同一层展开，而不能通过深度嵌套表达更细的分工粒度。

**3. 子 agent 无法与父 agent 通信**

子 agent 的 `clarify`、`send_message` 均被阻断。如果子 agent 执行中途需要澄清或遇到歧义，只能在最终 summary 中说明，父 agent 无法实时指导。这意味着 `context` 字段需要在委托前提供足够完整的信息。

**4. MoA 强依赖 OpenRouter**

`mixture_of_agents_tool` 硬依赖 `OPENROUTER_API_KEY`，所有参考模型和聚合模型均通过 OpenRouter 路由。在 Hermes 以直连 Anthropic/自定义端点方式运行时，MoA 不可用。参考模型列表（`REFERENCE_MODELS`）是模块级常量，无法通过 `config.yaml` 动态配置。

**5. tool_trace 信息有限**

父 agent 收到的 `tool_trace` 只记录工具名和参数字节数（`args_bytes`、`result_bytes`），不含工具调用内容和结果内容。父 agent 在制定下一步计划时对子 agent 的实际执行路径几乎不可见，只能依赖 summary 的自然语言描述。

**6. 批量任务数上限固定为 3**

`MAX_CONCURRENT_CHILDREN = 3` 且 `task_list = tasks[:MAX_CONCURRENT_CHILDREN]` 硬截断，超出部分直接丢弃（无报错）。对于需要并行 4+ 个任务的场景，调用方需要自行分批。

## 来源

- 源码版本：hermes-agent 0.16.0
- 分析文件：`tools/delegate_tool.py`、`run_agent.py`、`tools/mixture_of_agents_tool.py`、`cli-config.yaml.example`
- 分析深度：源码级
