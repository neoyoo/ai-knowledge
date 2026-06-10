---
title: "Tool System — Hermes Agent"
category: L2
parent: "[[tool-system]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的工具系统围绕一个模块级单例注册表（`ToolRegistry`）构建，采用"工具文件自注册"模式：每个 `tools/*.py` 文件在模块被 import 时主动调用 `registry.register()`，将自身的 schema、handler、check_fn 等元数据写入注册表。`model_tools.py` 负责触发全量 import，并对外暴露过滤、分发、类型强制等功能。Toolset 系统在工具集合层面进行分组管理，Distribution 系统进一步为批量运行和 RL 训练提供带权重的随机采样能力。整套架构的核心特征是：**注册与使用完全分离**——工具文件只知道如何注册自己，不感知调用链；分发层只知道如何路由，不感知工具内部逻辑。

## 架构分析

### 注册表设计：ToolEntry 元数据对象

`tools/registry.py` 定义了 `ToolEntry`，使用 `__slots__` 固定字段集合，包含：

| 字段 | 类型 | 说明 |
|------|------|------|
| `name` | str | 工具唯一标识，即模型 function call 中使用的名称 |
| `toolset` | str | 所属工具集，用于 toolset 级过滤 |
| `schema` | dict | OpenAI function-calling 格式的 JSON Schema |
| `handler` | Callable | 实际执行函数，同步或异步均可 |
| `check_fn` | Callable | 运行时可用性检查，返回 bool；None 表示无条件可用 |
| `requires_env` | list | 所需环境变量列表，用于诊断工具不可用的原因 |
| `is_async` | bool | 标记 handler 是否为 coroutine，由 dispatch 层自动 bridge |
| `description` | str | fallback 描述，优先从 schema 取 |
| `emoji` | str | 终端 UI 展示用 emoji |
| `max_result_size_chars` | int\|float\|None | 结果持久化阈值；`float('inf')` 表示永不截断 |

`ToolRegistry` 是模块级单例（`registry = ToolRegistry()`），在 `tools/registry.py` 底部直接实例化。所有工具文件 import `from tools.registry import registry` 后直接调用 `registry.register()`，在模块加载时完成注册，无需任何中心化配置文件。

典型注册调用（`tools/terminal_tool.py`）：

```python
registry.register(
    name="terminal",
    toolset="terminal",
    schema=TERMINAL_SCHEMA,
    handler=_handle_terminal,
    check_fn=check_terminal_requirements,
    emoji="💻",
    max_result_size_chars=100_000,
)
```

典型注册调用（`tools/file_tools.py`，展示 `max_result_size_chars=float('inf')` 防截断设计）：

```python
registry.register(name="read_file", toolset="file", schema=READ_FILE_SCHEMA,
    handler=_handle_read_file, check_fn=_check_file_reqs, emoji="📖",
    max_result_size_chars=float('inf'))
```

### 工具发现：_discover_tools() 导入链

`model_tools.py` 的 `_discover_tools()` 维护一个明确的模块名列表（共 20 个内建模块），用 `importlib.import_module()` 逐一加载，每个加载操作会触发该文件底部的 `registry.register()` 调用。关键设计：每个模块的加载都被 `try/except` 包裹，单个工具模块加载失败（如缺少可选依赖 `fal_client`）不会阻断其余工具加载。

`_discover_tools()` 完成后，紧接着有两条独立的发现路径：

1. **MCP 工具发现**：`from tools.mcp_tool import discover_mcp_tools; discover_mcp_tools()`——从 `~/.hermes/config.yaml` 的 `mcp_servers` 读取外部 MCP server 配置，动态注册；
2. **Plugin 工具发现**：`from hermes_cli.plugins import discover_plugins; discover_plugins()`——扫描用户目录、项目目录、pip entry-points 三个来源，调用插件的 `register(ctx)` 方法。

这三条发现路径均在 `model_tools.py` 模块加载时（即第一次 import `model_tools` 时）自动执行，保证后续所有调用者拿到的注册表是完整的。

### 工具可用性过滤：check_fn 机制

`registry.get_definitions()` 在返回 OpenAI 格式 schema 前会执行 check_fn 过滤。同一 check_fn 在一次调用中只执行一次（`check_results: Dict[Callable, bool]` 缓存），避免同一 toolset 的多个工具重复调用相同的环境检查。check_fn 抛出异常时视为不可用，不向上传播。

以 `web_tools.py` 的 `check_web_api_key` 为例：它检查是否有 Exa/Firecrawl/Tavily 等 web backend 的 API key 配置，如果没有则 `web_search` 和 `web_extract` 不会出现在 model 看到的工具列表里。

### 工具分发：handle_function_call() 的完整路径

`model_tools.py` 的 `handle_function_call()` 是分发主入口，执行以下步骤：

1. **类型强制**（`coerce_tool_args()`）：LLM 频繁将数字/布尔值以字符串形式返回（如 `"42"` 而非 `42`），`coerce_tool_args` 对照工具的 JSON Schema 中各字段的 `"type"` 声明，在类型不符时自动强制转换（`_coerce_number`、`_coerce_boolean`），支持 union type（`"type": ["integer", "string"]`）。

2. **Read-loop 追踪通知**：若调用的工具不是 `read_file`/`search_files`，则通知 `file_tools` 的连续读取计数器重置，防止模型陷入无限读文件循环。

3. **Agent-loop 工具拦截**：`_AGENT_LOOP_TOOLS = {"todo", "memory", "session_search", "delegate_task"}` 中的工具需要 `run_agent.py` 级别的状态（TodoStore、MemoryStore 等），在此处返回错误 stub，由 agent loop 层提前拦截。

4. **Plugin pre_tool_call hook**：调用 `invoke_hook("pre_tool_call", ...)`，允许插件在工具执行前注入逻辑（审计、限速、覆盖等）。

5. **注册表分发**：调用 `registry.dispatch(function_name, function_args, task_id=..., user_task=...)`，`execute_code` 工具特殊处理，传入 `enabled_tools` 以限制沙箱内可用工具列表。

6. **Plugin post_tool_call hook**：调用 `invoke_hook("post_tool_call", ...)`，允许插件在工具执行后注入逻辑（日志、指标等）。

`registry.dispatch()` 内部负责处理 async bridge：若 `entry.is_async`，则调用 `_run_async(entry.handler(args, **kwargs))`；所有未捕获异常被 catch 为 `{"error": "..."}` JSON 字符串，保证分发层永远不向上抛出。

### Async Bridge：三路持久化事件循环

`model_tools.py` 实现了一套精心设计的 sync→async bridge（`_run_async()`），解决以下问题：`asyncio.run()` 每次创建并销毁一个事件循环，导致绑定到该循环的 `httpx`/`AsyncOpenAI` 客户端在 GC 时触发 `"Event loop is closed"` 错误。

三条分支按调用上下文切换：

| 场景 | 检测方式 | 策略 |
|------|----------|------|
| 已在 async 上下文中（gateway、RL env） | `asyncio.get_running_loop()` 返回运行中的 loop | 新建 `ThreadPoolExecutor(max_workers=1)` 线程，在其中 `asyncio.run(coro)`，timeout=300s |
| 工作线程（delegate_task 的线程池） | `threading.current_thread() is not threading.main_thread()` | 使用 `threading.local()` 存储的 per-thread 持久化 loop（`_get_worker_loop()`） |
| 主线程（CLI 正常路径） | 以上两条均不满足 | 使用进程级持久化 loop（`_get_tool_loop()`，`_tool_loop_lock` 保护） |

这一设计是"单一真相来源"（注释原文：`single source of truth for sync->async bridging`），registry.dispatch 调用 `from model_tools import _run_async` 也走同一路径。

### Schema 动态修补：available-tools 一致性

`get_tool_definitions()` 在返回前做两处 schema 动态修补，解决"model 看到工具描述中提到的工具名但实际不可用"问题：

1. **execute_code 沙箱工具列表修补**：`build_execute_code_schema(sandbox_enabled)` 重建 `execute_code` 的 schema，只列出当前 session 真正可用的沙箱内工具；

2. **browser_navigate 描述修补**：若 `web_search`/`web_extract` 不可用，从 `browser_navigate` 描述中删除"prefer web_search or web_extract"这句话，避免 model 幻觉调用不存在的工具。

### Toolset 系统：声明式分组与组合

`toolsets.py` 维护一个静态 `TOOLSETS` 字典，每个 toolset 有 `tools`（直接工具列表）和 `includes`（引用其他 toolset）两个字段，支持组合：

```python
"debugging": {
    "tools": ["terminal", "process"],
    "includes": ["web", "file"]
}
```

`resolve_toolset(name, visited)` 递归解析，用 `visited` set 同时实现环检测和 diamond 依赖去重（diamond 依赖不会重复收集工具）。

特殊别名 `"all"`/`"*"` 解析为注册表中所有 toolset 的并集，确保未来新增 toolset 自动生效。

Hermes 为每个部署入口点维护了专属 toolset：`hermes-cli`、`hermes-telegram`、`hermes-discord`、`hermes-slack`、`hermes-whatsapp`、`hermes-signal`、`hermes-email`、`hermes-acp`（编辑器集成，无交互 UI 工具）、`hermes-api-server`（HTTP 服务，无 clarify/send_message）等，所有平台共享 `_HERMES_CORE_TOOLS` 常量，修改一处同步生效。

`hermes-gateway` 是最顶层的聚合 toolset，通过 `includes` 引用所有 messaging platform toolset，形成一个 union。

Plugin toolset 通过 `_get_plugin_toolset_names()` 动态注入，在 `get_all_toolsets()`、`validate_toolset()`、`resolve_toolset()` 中均有 fallback 路径，静态字典与动态注册对外表现一致。

### Distribution 采样：batch/RL 用的概率模型

`toolset_distributions.py` 定义了一套用于批量数据生成和 RL 训练的工具集采样机制：

```python
DISTRIBUTIONS = {
    "science": {
        "toolsets": {
            "web": 94, "terminal": 94, "file": 94,
            "vision": 65, "browser": 50, "image_gen": 15, "moa": 10
        }
    },
    ...
}
```

`sample_toolsets_from_distribution(distribution_name)` 对每个 toolset 独立抛硬币（`random.random() * 100 < probability`），允许多个 toolset 同时被选中（非互斥）。当随机结果全部未命中时，有保底逻辑：选择概率最高的那个 toolset，保证至少有一个工具可用。

分布名称如 `"science"`、`"research"`、`"browser_tasks"`、`"terminal_tasks"` 等，明确对应不同任务领域的工具配置，让 RL 训练时的工具环境与真实场景对齐。

### 结果持久化：三层防溢出机制

`tools/tool_result_storage.py` 实现三层 context-window 防溢出机制：

**Layer 1（工具内部）**：各工具自行预截断输出（如 `search_files` 限制返回行数），是工具作者唯一能控制的层。

**Layer 2（单结果）**：`maybe_persist_tool_result()` 在工具返回后检查：若输出超过 `registry.get_max_result_size(tool_name)` 的阈值，通过 `env.execute()` 将完整输出写入沙箱 `/tmp/hermes-results/{tool_use_id}.txt`，context 中替换为 `<persisted-output>` 标签 + 预览片段 + 文件路径。model 可用 `read_file` 按需读取全文。

**Layer 3（Turn 聚合预算）**：`enforce_turn_budget()` 在一个 assistant turn 所有工具结果回收后执行：若总字符数超过 `DEFAULT_TURN_BUDGET_CHARS = 200_000`，按大小降序对未持久化的结果进行 spill，直到总量降至预算内。

`DEFAULT_RESULT_SIZE_CHARS = 100_000`，`read_file` 被 pin 为 `float('inf')`（避免 persist→read→persist 无限循环）。

阈值解析优先级：`PINNED_THRESHOLDS` > `BudgetConfig.tool_overrides` > `registry.get_max_result_size()` > `DEFAULT_RESULT_SIZE_CHARS`。

### Tirith 安全扫描层

`tools/tirith_security.py` 为终端命令执行提供内容级深度扫描，是 `tools/approval.py` 正则匹配层之外的第二道防线，专门针对正则难以识别的语义威胁（同形字 URL、管道注入、终端转义注入等）。

**核心执行模型：**

```python
result = subprocess.run(
    [tirith_path, "check", "--json", "--non-interactive", "--shell", "posix", "--", command],
    capture_output=True, text=True, timeout=cfg["tirith_timeout"],
)
# 退出码是唯一的裁决来源
# 0 = allow, 1 = block, 2 = warn
# JSON stdout 仅用于补充 findings/summary，永远不覆盖退出码判定
```

tirith 以独立子进程运行，命令内容通过 `--` 分隔符传入，防止参数注入。退出码是裁决的唯一来源；JSON stdout 只提供 `findings`（最多 50 条）和 `summary`（最多 500 字符）用于日志记录，即使 JSON 解析失败，退出码裁决仍然有效。

**Fail-open 模型：**

`tirith_fail_open: true`（默认）——tirith 不可用时（spawn 失败、超时、未知退出码）一律放行，保证工具可用性优先。可通过 `TIRITH_FAIL_OPEN=false` 或 config 改为 fail-closed（更安全但影响可用性）。

**SHA-256 + cosign 供应链验证：**

tirith 二进制通过自动安装流程分两阶段验证：

| 阶段 | 工具 | 验证内容 |
|------|------|---------|
| 基础完整性 | SHA-256 (`hashlib`) | 对照官方 `checksums.txt`，验证下载文件未被篡改 |
| 供应链溯源 | cosign `verify-blob` | 验证 `checksums.txt` 由指定 GitHub Actions workflow 签名，证明发布来源于官方 CI |

cosign 验证使用固定的身份规则：
```python
_COSIGN_IDENTITY_REGEXP = "^https://github.com/sheeki03/tirith/.github/workflows/release.yml@refs/tags/v"
_COSIGN_ISSUER = "https://token.actions.githubusercontent.com"
```

cosign 不在 PATH 时自动降级为 SHA-256 only（HTTPS + 校验和仍提供传输层完整性），而不是阻断安装。cosign 验证明确失败（exit code ≠ 0）时中止安装。

**自动安装到 `$HERMES_HOME/bin/tirith`：**

解析优先级：
1. 用户显式配置的路径（不触发自动下载）
2. `shutil.which("tirith")`（PATH 查找）
3. `$HERMES_HOME/bin/tirith`（上次自动安装的缓存）
4. 从 GitHub Releases 自动下载（后台线程，不阻塞启动）

安装失败结果持久化到 `$HERMES_HOME/.tirith-install-failed` 文件（有效期 24 小时），避免每次命令执行都重试网络请求。特殊 case：失败原因为 `cosign_missing` 时，一旦 cosign 出现在 PATH，标记自动清除并触发重试，无需重启进程。

**与 approval.py 的关系：**

| 层次 | 位置 | 检测方式 | 目标威胁 |
|------|------|---------|---------|
| Layer 1 | `tools/approval.py` | 正则 `DANGEROUS_PATTERNS` | `rm -rf`、fork bomb、pipe-to-shell 等语法模式 |
| Layer 2 | `tools/tirith_security.py` | tirith 子进程（语义分析） | 同形字 URL、终端注入、混淆命令等内容级威胁 |

两层互补：approval.py 在前（快，正则匹配），tirith 在后（慢，5s 超时，内容级分析），任一层 block 即阻止执行。

### 危险命令安全层

`tools/approval.py` 为 terminal 工具提供独立的安全审批层：

- **检测**：`detect_dangerous_command()` 对命令归一化（ANSI 剥离、Unicode NFKC 标准化、去除 null byte）后匹配 `DANGEROUS_PATTERNS` 正则列表（包含 `rm -rf`、fork bomb、pipe-to-shell、SQL DROP 等 20+ 模式）；
- **Smart 审批**：`approvals.mode: smart` 时调用辅助 LLM 判断真实风险，自动放行假阳性（如 `python -c "print('hello')"` 被 `-c flag` 规则命中但实际无害）；
- **会话/永久 allowlist**：使用 `contextvars.ContextVar` 保证多线程 gateway 场景下会话隔离；
- **阻塞式 gateway 审批**：gateway 场景下，agent 线程阻塞在 `threading.Event.wait(timeout=300)` 上，等待用户通过 `/approve` 命令响应，FIFO 队列支持并发子 agent 同时等待审批。

## 关键代码路径

- `tools/registry.py` — `ToolRegistry` 单例、`ToolEntry` 元数据对象、`register()`/`dispatch()`/`get_definitions()`，以及 `tool_error()`/`tool_result()` 序列化工具函数
- `model_tools.py` — `_discover_tools()`（工具发现）、`handle_function_call()`（主分发入口）、`coerce_tool_args()`（类型强制）、`_run_async()`（三路 async bridge）
- `toolsets.py` — `TOOLSETS` 字典、`resolve_toolset()`（递归解析，环检测）、`get_all_toolsets()`（含 plugin toolset 合并）
- `toolset_distributions.py` — `DISTRIBUTIONS` 字典、`sample_toolsets_from_distribution()`（独立概率采样 + 保底逻辑）
- `tools/tool_result_storage.py` — `maybe_persist_tool_result()`（Layer 2 单结果持久化）、`enforce_turn_budget()`（Layer 3 Turn 聚合预算）
- `tools/budget_config.py` — `BudgetConfig`（不可变预算配置）、`PINNED_THRESHOLDS`（`read_file=inf`）
- `tools/approval.py` — `detect_dangerous_command()`、`check_all_command_guards()`、`_smart_approve()`（LLM 审批）、`_ApprovalEntry`（阻塞式 gateway 审批队列）
- `tools/mcp_tool.py` — MCP server 发现与动态注册（`discover_mcp_tools()`、`registry.register()`/`registry.deregister()` 用于 `notifications/tools/list_changed`）
- `hermes_cli/plugins.py` — plugin 系统（三来源发现、`PluginContext.register_tool()`、`invoke_hook()`）

## 设计亮点

**自注册模式消除中心化配置**：每个工具文件在模块 import 时自注册，新增工具只需在 `_modules` 列表加一行，不需修改任何路由表。MCP 工具和 plugin 工具走相同 registry，对上层完全透明。

**三路 async bridge 保护缓存客户端生命周期**：`_run_async()` 针对 CLI 主线程、工作线程池、async gateway 三种不同的事件循环上下文分别维护持久化 loop，从根本上解决 `asyncio.run()` 销毁 loop 导致 `httpx`/`AsyncOpenAI` 客户端 GC 报错的问题。这是一个真实的工程痛点，设计本身也体现了对 Python 异步模型的深度理解。

**Schema 动态修补保证工具可见性一致性**：`execute_code` schema 和 `browser_navigate` 描述在 `get_tool_definitions()` 返回前动态修补，确保 model 看到的工具引用与实际可用工具列表完全一致，避免 model 幻觉调用不存在的工具。这个细节解决了一个常见的 hallucination 来源。

**Distribution 系统支持工具分布多样性训练**：独立概率采样（非互斥）让每个 batch 样本拥有不同的工具组合，模型在训练时接触到更多样的工具情境，泛化性强于固定工具集。保底逻辑（至少选一个）防止空工具集导致训练样本无效。

**三层防溢出保护可组合**：Layer 1 在工具内自控，Layer 2 按工具阈值处理单结果，Layer 3 按 turn 预算处理聚合结果。三层各自独立，可在 `BudgetConfig` 中逐层覆盖阈值，RL 环境可传入定制 `BudgetConfig` 适配不同的 context 长度约束。`read_file=inf` 的 pin 设计是防止"持久化文件 → 读文件溢出 → 再次持久化"死循环的关键。

**危险命令安全层分离关注点**：approval 逻辑完全在 `tools/approval.py` 中，terminal 工具只调用 `check_all_command_guards()`。Smart 审批（LLM 判断）优雅处理正则假阳性。Gateway 阻塞审批用 `threading.Event` + FIFO 队列实现，支持并发子 agent 场景，每个子 agent 有独立的 `_ApprovalEntry`，不会互相干扰。

## 局限性

**`_AGENT_LOOP_TOOLS` 硬编码拆分点**：`todo`、`memory`、`session_search`、`delegate_task` 被 hardcode 为"需要 agent loop 层处理"的工具，它们的 schema 在注册表中存在，但 `handle_function_call()` 返回 error stub。这意味着工具系统本身不知道哪些工具依赖外部状态，必须在两个地方维护这个语义：registry 里注册 schema，`_AGENT_LOOP_TOOLS` 里声明不可分发。工具增多后这个拆分点容易失同步。

**MCP 动态工具的 check_fn 和 is_async 语义局限**：MCP 工具在 `mcp_tool.py` 中通过 `registry.register()` 动态注册，check_fn 依赖 MCP server 的连接状态，is_async 固定为 True，但实际上 MCP 工具的可用性是更细粒度的（某个 server 可能只有部分工具失效），当前粒度是 per-server 级别。此外，MCP 工具的 `deregister()` 是为 `notifications/tools/list_changed` 设计的"nuke-and-repave"策略，在高频工具变更的 MCP server 下可能产生短暂的 schema 空窗期。

**Distribution 采样缺乏工具有效性验证**：`sample_toolsets_from_distribution()` 只用 `validate_toolset()` 验证 toolset 名称是否存在，但不调用 check_fn 验证 toolset 在当前环境中实际可用。batch 运行中可能采样到因 API key 缺失而实际不可用的 toolset，导致训练样本中出现"工具被调用但不可用"的 error pattern。

**三路 async bridge 的 timeout 硬编码**：`_run_async()` 在 "已在 async 上下文中" 分支使用 `future.result(timeout=300)` 硬编码 5 分钟 timeout，对于长时间运行的工具（如大文件处理、慢速 MCP server）可能不够，但也无法从工具层传入 per-tool timeout，统一 timeout 是粗粒度的安全网。

## 来源

- 源码版本：hermes-agent 0.16.0
- 分析深度：源码级
- 主要分析文件：`tools/registry.py`、`model_tools.py`、`toolsets.py`、`toolset_distributions.py`、`tools/tool_result_storage.py`、`tools/budget_config.py`、`tools/approval.py`、`tools/mcp_tool.py`、`hermes_cli/plugins.py`、`tools/terminal_tool.py`、`tools/web_tools.py`、`tools/file_tools.py`、`tools/delegate_tool.py`
