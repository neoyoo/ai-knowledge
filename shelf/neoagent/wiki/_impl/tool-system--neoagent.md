---
title: "tool-system — neoagent"
category: L2
parent: "[[tool-system]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: tool-system
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的工具系统由 4 个解耦组件组成：`BaseTool`（抽象基类）→ `ToolRegistry`（纯数据结构注册表）→ `ToolExecutor`（执行层，管权限/并发/截断/hook/事件）→ `DeferredToolRegistry`（延迟索引，专为 MCP 用）。特色是 `BaseTool.auto_free_after: int` 字段——工具声明"我的结果超过 N 轮后应被折叠"，使上下文生命周期管理进入工具协议层；内置工具共 13 个，含 6 个文件操作、`run_python` 沙盒、`skill_load`/`write_skill` 动态技能、`tool_search`（MCP 发现）、`free_tool_result`/`recall_tool_result`（LLM 主动上下文管理）。

## 架构分析

### BaseTool 协议（六字段）

```python
class BaseTool(ABC):
    name: str
    description: str
    input_model: type[BaseModel]             # Pydantic 模型，用来自动生成 JSON Schema
    permission: Literal["auto","ask","deny"] = "ask"
    is_concurrent_safe: bool = False
    auto_free_after: int = 0                 # 0 = 永不；N = N 轮后自动 free
    
    @abstractmethod
    async def execute(self, input: BaseModel) -> ToolResult: ...
    def preview(self, output: str) -> str:   # 80-char 预览，多媒体工具可覆写
        ...
    def get_schema(self) -> dict:
        raw = self.input_model.model_json_schema()
        raw.pop("title", None)
        return {"name": self.name, "description": self.description, "input_schema": raw}
```

默认 `permission="ask"`（fail-closed），`is_concurrent_safe=False`（fail-closed），`auto_free_after=0`（fail-open，保守不 free）。

### ToolRegistry：纯数据结构

31 行实现，只做 dict 管理 + 过滤 deny。`get_schemas()` 返回 `permission != "deny"` 的工具列表——被 deny 的工具连 schema 都不暴露给 LLM（vs 只有执行时拒绝）。`all_tools()` 返回副本 dict。`unregister(name)` 无副作用。

### ToolExecutor：执行层

159 行，负责：
1. **并发分区**：`_partition_by_concurrency(calls)` 按 `tool.is_concurrent_safe` 把 calls 分成两组；safe 组走 `asyncio.gather(*[self._run_one(idx, call) for ...])` 并行，unsafe 组 `for + await` 串行。结果用 idx 回填保证顺序。
2. **ToolCallEvent 发射**：`input_data` 用 `MappingProxyType(copy.deepcopy(call.input))` 包装，防止 handler mutate。
3. **Pydantic 校验**：`tool.input_model.model_validate(call.input)` 抛 ValidationError 进 except。
4. **PermissionChecker.check**：`permission="auto"` 直通，`"deny"` 永远 False，`"ask"` 走 ask_callback 或 auto_approve（`auto_approve=True` 时印 WARNING 后通过）。
5. **pre_tool_call hook**（在权限检查之后）：deny 短路；modify 替换 tool_input 后**重新走权限检查**——注释明写"hooks must not be able to bypass PermissionChecker by substituting a safe input with a dangerous one after the initial check passed"。
6. **tool.execute(validated_input)** → `ToolResult`，按 `max_result_size`（默认 50000）截断（在 output 尾部追加 `\n[truncated]`）。
7. **post_tool_call hook**：可 modify result，ignored deny。
8. **ToolResultEvent 发射**。

### DeferredToolRegistry：MCP 专属

123 行，独立于 ToolRegistry 维护 `_all: dict[str, ToolIndex]` 和 `_deferred: set[str]`。注册时同时加入两个容器；`promote(names)` 从 _deferred 移除使 schema 下一轮可见；`search(query)` 支持三种模式：
- `"select:name1,name2"` — 精确匹配
- `"+keyword"` — name 必须包含 keyword（case-insensitive）
- 其他 — `re.compile(pattern, IGNORECASE)` 跨 name+description 搜索（pattern > 200 字截断；re.error 降级为 `re.escape()` 子串匹配）

### 内置工具特色

| 工具 | 关键参数 | 独特点 |
|------|----------|--------|
| `run_python` | `auto_free_after=2`, permission="auto" | `asyncio.create_subprocess_exec(sys.executable, "-c", code)`，`start_new_session=True` 以便 SIGKILL 进程组；env 白名单仅保留 PATH/HOME/HTTP_PROXY 等 9 项；超时 `os.killpg(proc.pid, SIGKILL)` |
| `tool_search` | permission="auto", is_concurrent_safe=True | 调 deferred_registry.search → 在 ToolRegistry 验证存在（跳过 ghost）→ 写入 `session_state.promoted_tools`（ContextVar 取得）+ deferred.promote → 只返回 name+description（不返回 full schema，schema 下一轮才可见） |
| `free_tool_result` | permission="auto" | 从 ContextVar 取 `_current_session` → 遍历 messages 找目标 ToolResultBlock → 复制 content 到 `FreedToolResult(id, tool_name, size, preview, original_content)` → 加入 `session.state.freed_tool_results` |
| `recall_tool_result` | `auto_free_after=1`, permission="auto" | 同 session 获取 → 查 `session.state.freed_tool_results[id]` → 写入 `recalled_this_turn` 集合（loop 的 `_apply_freed_to_messages` 会跳过）→ 返回 `original_content`；**`auto_free_after=1` 保证下一轮 recall 结果自动回到 freed 状态，不会永久 re-inflate** |
| `skill_load` | permission="auto" | 支持三种 layout：`skills_dir/{name}.md`、`skills_dir/{name}/SKILL.md`、`skills_dir/learned/{name}/SKILL.md`；`learned/` 下的 skill 永远绕过 allowed 白名单（注释：agent 自己写的，总是安全） |
| `write_skill` | permission="auto" | kebab-case 校验；写到 `learned/{skill_name}/SKILL.md`；frontmatter 用 `name + description` 两字段 |

### 关键代码路径

- `neoagent/tools/base.py:8-31` — BaseTool 协议
- `neoagent/tools/registry.py:5-31` — ToolRegistry
- `neoagent/tools/executor.py:40-55` — 并发分区 + gather 并发执行
- `neoagent/tools/executor.py:93-124` — pre_tool_call hook 后的二次权限检查
- `neoagent/tools/deferred.py:57-71` — `search()` 三模式路由
- `neoagent/tools/builtin/run_python.py:92-140` — sandbox env 白名单 + killpg 超时
- `neoagent/tools/builtin/tool_search.py:83-117` — promote 到 session state + 只返 name/description
- `neoagent/agent.py:82-83` — `register_tool()` = `_registry.register(tool)`
- `neoagent/agent.py:320-369` — `add_mcp_server()`：StdioTransport + MCPClient + create_mcp_tools 循环注册到两个 registry，首个 server 时自动注册 `ToolSearchTool`

## 设计亮点

- **`auto_free_after` 写在 BaseTool 协议层**：让工具自己声明结果生命周期（run_python 2 轮后 free、recall 1 轮后 free、普通工具永不），不需要 QueryLoop 猜测。这是 neoagent 原创设计，和 Claude Code 的 "应用后自动遗忘" 思路接近但形式化到协议层。
- **pre_tool_call hook 后的二次权限校验**：防止 hook modify 替换安全 input 为危险 input，是典型的 defense-in-depth。
- **ToolSearch 只返回 name+description，schema 下一轮才可见**：相比一次 tool_search 把 full schema 打给 LLM，节约 token；同时保持"调用某工具前必须先 search 并等待下一轮"的显式 handshake。
- **free/recall 成对设计**：freed 默认持久化 original_content 在 session state 里（所以 recall 可以拿回原文），但 provider 收到的是 placeholder——这种"provider 视图 vs session 真相"分离让 LLM 觉得信息消失了，但工具层确实能恢复，是非常精巧的 upcycling 设计。
- **学习型 skill（learned/）绕过 allowed 白名单**：agent 自己 `write_skill` 写的文件不受外部配置限制，构建了"agent 自我进化"的闭环。
- **env 白名单 + killpg 的 run_python 沙盒**：env 只传 PATH/HOME/HTTP_PROXY 等 9 个白名单键（不传 API_KEY），`start_new_session=True` 让超时时能一键杀进程组——相比裸 asyncio.subprocess 多走了两步安全加固。

## 局限性

- **只有"声明式 concurrent-safe"，没有路径级判定**：和 Hermes Agent 的 `_paths_overlap()` 路径级安全判定相比粗，两个都写 `is_concurrent_safe=False` 的工具永远串行，哪怕它们操作不同文件。
- **工具schema 生成时 `raw_schema.pop("title", None)` 是唯一净化**：Pydantic 生成的 JSON Schema 如果含 `$defs`/`anyOf` 等，会完整透传给 provider，可能在某些 provider（如 legacy OpenAI function calling）出问题——neoagent 未做 schema 转换层。
- **`max_result_size=50000` 是全局配置**：无 tool 级覆盖，大输出工具（run_python、fetch_html）和小工具（list_files）共享同一阈值。
- **DeferredToolRegistry 的 ghost entry 问题**：remove_mcp_server 会清理，但如果 client.connect 失败的半状态，可能留下 DeferredToolRegistry 有但 ToolRegistry 没有的 ghost——`tool_search.execute` 里有 `if tool is None: ... skipping` 兜底，但仍是"能跑但状态不一致"的设计。
- **ContextVar 传递 session 是全局 module 级**：`_current_session` 是 `tools.builtin.tool_search` 的 ContextVar——意味着 free/recall/tool_search 三个工具物理上依赖同一个 module，不能拆分包；对未来把 builtin tools 改为独立 package 是隐患。
- **没有工具依赖图/组合工具**：每个工具独立，没有 AgentScope 的 ToolGroup 或 Hermes 的 Toolset 声明式分组概念。
- **`permission="deny"` 工具连 schema 都不暴露**：`get_schemas()` 只返回非 deny 的，意味着运行时切换一个工具为 deny 需要下一轮才反映到 LLM——中途状态切换无保证。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/tools/base.py`, `registry.py`, `executor.py`, `deferred.py`, `permission.py`, `builtin/*.py`
