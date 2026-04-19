---
title: "mcp-skills — neoagent"
category: L2
parent: "[[mcp-skills]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: mcp-skills
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 同时实现 MCP 集成和 Skill 系统两套扩展机制。**MCP 侧**：`MCPClient`（JSON-RPC 2.0 over stdio，含乱序响应缓冲）+ `MCPTool`（JSON Schema → Pydantic 递归转换，支持 array/object/enum）+ `DeferredToolRegistry`（MCP 工具注册但默认隐藏，通过 `tool_search` 按需提升）+ `add_mcp_server()` 连接 lifecycle。**Skill 侧**：`SkillLoadTool`（progressive disclosure，skill_dir + allowed 白名单 + learned/ 绕过白名单机制）+ `WriteSkillTool`（agent 自我成长：写 learned/ 目录产生新 skill）+ `PromptBuilder.register/activate_skill` 三态管理。独特设计：**session-scoped MCP 提升**（`session_state.promoted_tools` + 全局 `is_deferred()` OR 逻辑）让不同 session 独立看到不同子集、**learned skill 绕过 allowed 白名单**支持 agent 自写自用的成长闭环。

## 架构分析

### MCP 两层注册模型

`add_mcp_server(name, command, env)` 流程：

```python
# neoagent/agent.py:320-369
async def add_mcp_server(self, name, command, env=None):
    if not command: raise ValueError
    if re.search(r'[;&|`$(){}]', command[0]):
        raise ValueError(f"shell metacharacters: {command[0]!r}")
    
    transport = StdioTransport(command=command, env=env)
    client = MCPClient(name=name, transport=transport)
    await client.connect()  # subprocess + MCP initialize handshake
    
    tools = await create_mcp_tools(client=client, server_name=name)
    for tool in tools:
        self._registry.register(tool)             # ToolRegistry（执行用）
        self._deferred_registry.register(tool.name, tool.description)  # DeferredToolRegistry（搜索用）
    
    self._mcp_clients[name] = client
    
    # First MCP server: auto-register tool_search
    if len(self._mcp_clients) == 1:
        tool_search = ToolSearchTool(
            deferred_registry=self._deferred_registry,
            tool_registry=self._registry,
        )
        self._registry.register(tool_search)
```

关键点：
- **Shell injection 防护**：`re.search(r'[;&|`$(){}]', command[0])` 检查 binary 名——虽然 `subprocess_exec` 不经 shell 不易注入，仍显式阻断 `"npx;evil"` 之类的可疑串。
- **双 registry 注册**：工具进 ToolRegistry（提供给 ToolExecutor 执行）的同时进 DeferredToolRegistry（提供 `search/promote`）——两个 registry 同名 key 指向同一逻辑工具但职责分离。
- **延迟到首个 MCP server 自动注册 `tool_search`**：不用 MCP 的用户不会看到 tool_search 工具，避免污染 tool list。
- **`remove_mcp_server(name)`** 用 `f"{name}__"` 前缀找工具，从 ToolRegistry 和 DeferredToolRegistry 同步清理——防止 ghost entry。

### DeferredToolRegistry：三模式搜索

```python
class DeferredToolRegistry:
    def __init__(self):
        self._all: dict[str, ToolIndex] = {}      # 所有注册过的
        self._deferred: set[str] = set()           # 未 promote 的
    
    def register(self, tool_name, description):   # 同时写入 _all 和 _deferred
    def promote(self, tool_names):                 # 从 _deferred 移除，_all 不变
    def is_deferred(self, tool_name):              # 是否 still 隐藏
    def search(self, query):                       # 三模式
```

`search` 三种查询：
- `"select:name1,name2"` → 精确匹配，只返 deferred 的
- `"+keyword ..."` → name 必须包含 keyword（case-insensitive）
- `"regex"` → `re.compile(pattern, IGNORECASE)` 跨 name+description 搜索；pattern > 200 字截断；`re.error` 降级 `re.escape` 字面子串

### `tool_search` 工具的 session-scoped 提升

```python
# neoagent/tools/builtin/tool_search.py
_current_session_state: ContextVar[SessionState | None] = ContextVar(
    "_current_session_state", default=None
)

class ToolSearchTool(BaseTool):
    async def execute(self, input: ToolSearchInput) -> ToolResult:
        query = input.query.strip()
        matches = self._deferred.search(query)
        
        promoted_names: set[str] = set()
        results: list[dict] = []
        for idx in matches:
            tool = self._registry.get_tool(idx.name)
            if tool is None:
                continue   # ghost entry, skip
            results.append({"name": idx.name, "description": idx.description})
            promoted_names.add(idx.name)
        
        if promoted_names:
            ss = _current_session_state.get()
            if ss is not None:
                ss.promoted_tools.update(promoted_names)   # session 级
            self._deferred.promote(promoted_names)          # 全局级（backward compat）
        
        return ToolResult(output=json.dumps(results))  # 只返回 name+description，不返 schema
```

**`QueryLoop.run()` 里的 OR 逻辑**（`core/loop.py:131-149`）：

```python
def _is_hidden(tool_name):
    if tool_name not in deferred_registry._all:
        return False  # not managed
    return (
        tool_name not in session_state.promoted_tools
        and deferred_registry.is_deferred(tool_name)
    )
# 两个条件都为 hidden 才真的隐藏；任一 promoted 就可见
```

这使 FastAPI 并发请求各自维护独立的 promoted_tools，不会污染邻居的 session；但保留全局 is_deferred 作为 backward-compat fallback。

### JSON Schema → Pydantic 递归转换

`mcp/tool.py:_json_schema_to_type` 递归实现：

- `{"enum": [...]}` → `Literal[val1, val2]`
- `{"type": "string"}` → `str`
- `{"type": "integer"}` → `int`
- `{"type": "number"}` → `float`
- `{"type": "boolean"}` → `bool`
- `{"type": "array", "items": <schema>}` → `list[<递归类型>]`
- `{"type": "object", "properties": {...}}` → 嵌套 Pydantic model（`_unique_model_name` 全局计数器保证唯一）
- fallback → `Any`

`_build_pydantic_model` 根据 `required` 决定字段是 `(py_type, Field(..., description=...))`（必填）或 `(py_type | None, Field(None, ...))` 或 `(py_type, Field(default, ...))` 含默认。

这样 MCP 工具可以走和本地 BaseTool 完全一致的 `input_model.model_validate()` 路径——Pydantic 校验错误被 ToolExecutor 统一 catch 成 ToolResult(is_error=True)。

### MCP JSON-RPC 协议实现

`MCPClient._request(method, params)` 关键点：

```python
async def _request(self, method, params) -> dict:
    req_id = self._next_request_id()
    
    # 检查提前到达的响应
    if req_id in self._pending:
        return self._pending.pop(req_id)
    
    await self._transport.send({"jsonrpc": "2.0", "id": req_id, "method": method, "params": params})
    
    while True:
        msg = await self._transport.receive()
        if "id" not in msg:
            logger.debug("notification %s, skipping", msg.get("method"))
            continue                          # Notification，跳过
        if msg["id"] == req_id:
            return msg                        # 匹配当前请求
        # 乱序响应，缓冲
        self._pending[msg["id"]] = msg
```

**乱序响应缓冲**`_pending` 解决 MCP server 可能不按请求顺序返回响应的问题——在复用单 transport 多个并发请求的场景关键（不过当前代码未见并发使用）。

### Skill 系统

`SkillLoadTool`（`builtin/skill_load.py`）：

```python
class SkillLoadTool(BaseTool):
    def __init__(self, skills_dir, allowed=None):
        self._skills_dir = Path(skills_dir)
        self._allowed = set(allowed) if allowed is not None else None
    
    def list_skills(self) -> list[dict]:
        # 三种 layout 都扫：
        # 1. skills_dir/{name}.md
        # 2. skills_dir/{name}/SKILL.md
        # 3. skills_dir/learned/{name}/SKILL.md     ← agent 自己写的
        skills = []
        for path in sorted(skills_dir.rglob("*.md")):
            name, desc, _ = _parse_skill_file(path)
            is_learned = "learned" in path.parts and path.parts[-2] != skills_dir.name
            if not is_learned and self._allowed is not None and name not in self._allowed:
                continue                        # learned 绕过白名单
            skills.append({"name": name, "description": desc})
        return skills
    
    async def execute(self, input):
        # 检查 allowed 白名单（learned 绕过）
        if self._allowed is not None and input.skill_name not in self._allowed \
           and not _is_learned_skill(self._skills_dir, input.skill_name):
            return error
        skill_file = _find_skill_file(self._skills_dir, input.skill_name)
        if skill_file is None:
            return error
        _, _, content = _parse_skill_file(skill_file)
        return ToolResult(output=content)        # 返回 skill 完整 markdown
```

Frontmatter 格式：

```yaml
---
name: xiumi-pattern
description: Extract article text from Xiumi image-layout pages via embedded JSON
---

# 适用条件
...
```

`WriteSkillTool`（`builtin/write_skill.py`）：

```python
class WriteSkillTool(BaseTool):
    async def execute(self, input):
        # kebab-case 校验
        if not _KEBAB_RE.match(input.skill_name):
            return error("Must be kebab-case...")
        
        skill_dir = self._learned_dir / input.skill_name
        skill_dir.mkdir(parents=True, exist_ok=True)
        skill_file = skill_dir / "SKILL.md"
        
        frontmatter = f"---\nname: {input.skill_name}\ndescription: {input.description}\n---\n\n"
        skill_file.write_text(frontmatter + input.content)
        return ToolResult(output=json.dumps({"status": "written", "path": str(skill_file), "name": input.skill_name}))
```

**只能写 `learned_dir`**（一般是 `skills_dir/learned/`），不能覆盖用户预定义 skill——安全隔离。配合 `list_skills` 的 learned 绕过白名单：**agent 写的 skill 下次自动可见可用**，无需外部配置变更。

### PromptBuilder 的 Skill 三态

```python
builder.register_skill("xiumi-pattern", PromptSection(name=..., content=..., priority=5))
                                        # 已注册但未 active
builder.activate_skill("xiumi-pattern")  # 加入 _sections，下次 build() 出现
builder.deactivate_skill("xiumi-pattern")
```

目前 QueryLoop 不自动调这三个 API——应用代码负责把 `skill_load` 调用的返回结果包成 PromptSection 并手动 activate。

### 关键代码路径

- `neoagent/agent.py:320-369` — `add_mcp_server` + shell injection 防护
- `neoagent/agent.py:371-407` — `remove_mcp_server` / `list_mcp_servers` / `close`
- `neoagent/mcp/client.py:44-93` — `_request` + 乱序缓冲
- `neoagent/mcp/tool.py:23-103` — JSON Schema 递归转换
- `neoagent/mcp/tool.py:117-166` — `MCPTool` 包装 + `create_mcp_tools`
- `neoagent/tools/deferred.py:16-122` — DeferredToolRegistry 三模式搜索
- `neoagent/tools/builtin/tool_search.py:51-117` — tool_search 工具 + session-scoped 提升
- `neoagent/tools/builtin/skill_load.py:76-135` — SkillLoadTool + learned 绕过
- `neoagent/tools/builtin/write_skill.py:21-95` — WriteSkillTool + kebab-case 校验
- `neoagent/core/prompt.py:25-43` — register_skill / activate_skill / deactivate_skill

## 设计亮点

- **MCP 工具默认 deferred + tool_search 按需提升**：大量 MCP server 连上时（十几个服务器几百个工具）不会一次性 flood LLM 的 tool list——LLM 只看到 `tool_search` 一个工具，用关键词查询后再加载所需子集。和 DeerFlow 的 DeferredToolRegistry 同思路，neoagent 额外做了 session-scoped 提升（并发隔离）。
- **Session-scoped promoted_tools**：`SessionState.promoted_tools` 让 FastAPI 并发请求各自独立 MCP 可见范围——前一个请求 promote 了 github_* 工具，不污染邻居 session。
- **`tool_search` 只返回 name+description 不返 schema**：节约当轮 token；schema 下一轮才通过 `ToolRegistry.get_schemas()` 出现——强制"search 之后等待下一轮"的 handshake。
- **JSON Schema 递归转 Pydantic**：对任意深度嵌套 MCP schema 都能生成真实的 Pydantic model，走和本地工具一致的 `model_validate` 路径——统一错误处理。
- **Learned skill 绕过 allowed 白名单**：agent 的 `write_skill` 产生的文件下次自动可见可用，构建 **"agent 自我扩展能力包"** 的闭环。这和 Anthropic Claude Code 的 skill-learning 机制思路接近。
- **Kebab-case 校验 + 目录隔离**：`WriteSkillTool` 只能写到 `learned/{name}/SKILL.md`，不能覆盖用户预定义 skill——agent 不会"偷偷覆盖"重要的配置文件。
- **Shell metacharacter 检查**：`add_mcp_server` 对 binary 名做静态检查——防护弱（binary 名一般无 shell meta）但体现"security by default"态度。
- **MCP `_pending` 乱序响应缓冲**：按 JSON-RPC 2.0 规范处理响应乱序——健壮性设计。
- **Notification 跳过**：MCP 服务器可能发 `notifications/initialized`、`notifications/progress` 等无 id 消息，`_request` 循环识别后跳过不影响请求-响应匹配。

## 局限性

- **只有 StdioTransport**：远程 MCP server（HTTP+SSE）不支持——`transport.py` 注释 "Reserved for future: SseTransport, not implemented"。要集成云端 MCP 必须包装成本地 stdio subprocess proxy。
- **Skill prompt injection 无自动化**：`skill_load` 返回 markdown 后，**不会自动 `register_skill` + `activate_skill`**；应用代码必须自己写适配逻辑 —— 文档和集成不够开箱即用。
- **Learned skill 无版本 / sunset**：`write_skill` 只有"写"，没有"删"/"更新"接口；agent 写错 skill 后，下次还会被 skill_load 看到。
- **MCPTool.permission 默认 "ask"**：每个 MCP 工具调用都要用户确认，非交互场景必须全局 `auto_approve=True`——太粗；没有"这个 server 的工具自动批准、那个必须确认"的 per-server 配置。
- **SkillChangeEvent 未发射**：`PromptBuilder.register/activate/deactivate_skill` 不发 event，Observer 看不到 skill 生命周期——audit / debug 不便。
- **JSON Schema 转 Pydantic 不处理 `oneOf/anyOf/$ref`**：fallback 到 `Any`——复杂 schema 丢失校验能力。
- **MCP 子进程 crash 无自动重连**：`MCPClient._request` 从 transport 收到 ConnectionError 时抛异常，没有 reconnect——agent 层必须手动 close + re-add_mcp_server。
- **`ToolSearchTool` 一次提升所有 match 结果**：没有"Top-N 结果"限制，regex 搜到 50 个工具会一次全部 promote 到 session——token 预算和选择噪声陡增。
- **`_deferred.search` 里 `len(pattern) > 200` 截断是 DoS 防护**：避免用户构造超大 regex 炸 Python regex 引擎——好但文档未说明。
- **多 MCP server 工具名冲突**：`{server_name}__{tool_name}` 前缀规避，但如果两个 server 都叫 "github"，第二次 `add_mcp_server("github", ...)` 会和第一个冲突（ToolRegistry 注册 ValueError）——框架不做 name 自动去重。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/agent.py:320-407`, `neoagent/mcp/{client,tool,transport}.py`, `neoagent/tools/deferred.py`, `neoagent/tools/builtin/{tool_search,skill_load,write_skill}.py`, `neoagent/core/prompt.py:25-43`
