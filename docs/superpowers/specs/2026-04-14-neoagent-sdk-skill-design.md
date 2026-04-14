# neoagent SDK Skill 设计文档

> 日期: 2026-04-14
> 状态: Draft
> 作者: Neo + Claude

---

## 1. 目标

构建一个 Claude Code skill，让 LLM 能够：

1. **写新 agent 代码**：拿到需求后，选择正确的 neoagent 模块组合，写出可运行的代码
2. **读懂/调试现有 agent**：看到 neoagent 代码时，理解模块间的数据流和调用链

Skill 位置：`~/.claude/skills/neoagent/SKILL.md`（全局安装，任何项目可调用）

触发描述：
> "Use when building agents with the neoagent SDK, understanding how SDK modules connect, adding capabilities (memory / custom tools / multi-agent / hooks / HTTP channel), or reading/debugging existing neoagent code."

---

## 2. Skill 结构（方案 C：混合型）

Skill 内嵌三块核心内容，深度细节指向外部文档：

| 块 | 内容 | 形式 |
|----|------|------|
| 模块接线图 | 全部 12 个模块如何串起来 | 文字架构图（嵌入） |
| 场景快查表 | 10 个场景 → 模块 + 关键 API | 表格（嵌入） |
| 最小代码骨架 | 每个场景的最小连线代码 | 代码块（嵌入） |
| 深度参考 | 需要细节时去哪里读 | 路径指针 |

---

## 3. 模块接线图

12 个模块完整覆盖，对应 wiki 12 页：

```
NeoAgentConfig (config.py)
    ↓
NeoAgent (agent.py)  ← 唯一入口：chat() / run()
    │
    ├── [prompt-system] PromptBuilder (core/prompt.py)
    │   ├── add_section()         永久 prompt 块（identity/memory）
    │   ├── register_skill()      注册 skill（不激活）
    │   └── activate_skill()      按需注入 skill → system prompt
    │
    ├── [query-loop] QueryLoop (core/loop.py)     ← 执行引擎（状态机）
    │   ├── [tool-system] Provider (providers/)    API 调用（Anthropic/OpenAI）
    │   ├── ToolExecutor
    │   │   ├── ToolRegistry                       活跃工具（LLM 可见）
    │   │   └── DeferredToolRegistry               工具池（默认隐藏）
    │   │       ↑ tool_search 内置工具
    │   ├── [hooks] HookManager (hooks.py)         前/后拦截
    │   └── [context-management] ContextCompressor (core/compress.py)  自动压缩
    │
    ├── [memory-system] MemoryManager (memory/)    跨会话记忆（memory_dir 启用）
    │   ├── MemoryExtractor               提取记忆
    │   ├── MemoryRetriever               检索记忆
    │   └── MemoryStore                   持久化
    │
    ├── [runtime-state / session-recovery] Session + SessionState (session.py)
    │   └── JsonFileStorage               持久化（session_dir 启用）
    │
    ├── [hooks] EventBus (events.py)      事件发布/订阅（可观测）
    └── [evaluation-observability] observe.py   终端输出 / 日志观察者

─── 独立扩展模块 ─────────────────────────────────

[mcp-skills] MCP (mcp/)       [multi-agent] Multi-Agent (multi/)
    MCPClient                     Orchestrator
    MCPTool                       WorkerCard
    MCPTransport                  register_worker()
    → 工具注入 ToolRegistry        await run(message)

[channel-remote] Channel (channels/)    [evaluation-observability] Eval (eval/)
    BaseChannel                           EvalRunner
    FastAPIChannel                        metrics / usage
    channel.run(host, port)               → 质量评估 + 指标采集
```

**wiki 模块 → neoagent 实现对照表：**

| wiki 模块 | neoagent 实现 |
|----------|--------------|
| query-loop | `core/loop.py` |
| prompt-system | `core/prompt.py` (+ Skills) |
| tool-system | `tools/` 整个子包 |
| context-management | `core/compress.py` |
| memory-system | `memory/` 整个子包 |
| runtime-state | `session.py` → SessionState |
| session-recovery | `session.py` → Session.resume() |
| hooks | `hooks.py` + `events.py` |
| mcp-skills | `mcp/` + DeferredToolRegistry |
| multi-agent | `multi/` |
| channel-remote | `channels/` |
| evaluation-observability | `eval/` + `observe.py` |

---

## 4. 场景快查表（10 个场景）

| 场景 | 核心模块 | 关键 API / 配置字段 |
|------|---------|-------------------|
| **1. 最小 agent** | NeoAgent + NeoAgentConfig | `api_key`, `model`, `system_prompt`, `max_turns`, `context_budget` |
| **2. 加自定义工具** | BaseTool → register_tool() | 继承 `BaseTool`，实现 `execute()`，设 `permission`；调 `agent.register_tool(tool)` |
| **3. 加跨会话记忆** | MemoryManager | `memory_dir=Path(...)`, `memory_project_key="..."` |
| **4. 加动态 Skill** | PromptBuilder | `agent._prompt_builder.register_skill(name, PromptSection(...))`→`activate_skill(name)` |
| **5. 加事件 Hook** | HookManager + EventBus | config 注册 hook；`agent._event_bus.subscribe(EventType, handler)` |
| **6. 接入 MCP 工具** | mcp/ + DeferredToolRegistry | `await agent.add_mcp_server(name, command, env)` → 工具自动注入 DeferredToolRegistry；LLM 用 `tool_search` 按需 promote |
| **7. 多 Agent** | Orchestrator + WorkerCard | `Orchestrator(config, max_depth=2)`，`register_worker(WorkerCard(...))`，`await run(msg)` |
| **8. HTTP 服务** | FastAPIChannel | `FastAPIChannel(agent)` → `await channel.run(host, port)` |
| **9. 断点续传** | Session + JsonFileStorage | `session_dir=Path(...)`；`Session.resume(session_id, storage)` |
| **10. 可观测性** | eval/ + observe.py | `Observer(console=True)` + `ObserverSubscriber.attach(agent._event_bus)`；`EvalRunner(agent).run(test_cases)` |

---

## 5. 最小代码骨架

### 场景 1：最小 agent
```python
from neoagent import NeoAgent, NeoAgentConfig

config = NeoAgentConfig(
    api_key="...",
    model="claude-sonnet-4-20250514",
    system_prompt="You are a helpful assistant.",
    max_turns=30,
    context_budget=80_000,
)
agent = NeoAgent(config)
result = await agent.chat("Hello")
```

### 场景 2：加自定义工具
```python
from neoagent.tools.base import BaseTool, ToolResult
from pydantic import BaseModel

class MyInput(BaseModel):
    query: str

class MyTool(BaseTool):
    name = "my_tool"
    description = "Does something useful"
    input_model = MyInput
    permission = "auto"          # auto / ask / deny
    is_concurrent_safe = True

    async def execute(self, input: MyInput) -> ToolResult:
        return ToolResult(call_id="", output=f"result: {input.query}")

agent.register_tool(MyTool())
```

### 场景 3：加跨会话记忆
```python
from pathlib import Path

config = NeoAgentConfig(
    ...,
    memory_dir=Path("~/.neoagent/memory").expanduser(),
    memory_project_key="my-project",
)
# MemoryManager 自动启用，无需手动操作
```

### 场景 4：加动态 Skill
```python
from neoagent.core.prompt import PromptSection

section = PromptSection(name="coding-skill", content="You are an expert Python developer...")
agent._prompt_builder.register_skill("coding-skill", section)

agent._prompt_builder.activate_skill("coding-skill")    # 激活
agent._prompt_builder.deactivate_skill("coding-skill")  # 关闭
```

### 场景 5：加事件 Hook
```python
from neoagent.events import EventType

# 订阅事件（观察）
agent._event_bus.subscribe(EventType.TOOL_EXECUTED, lambda e: print(e))

# 拦截 hook（HookManager 配置）→ 参考 docs/guide.md
```

### 场景 6：接入 MCP 工具
```python
# NeoAgent 直接管理 MCP server 生命周期
await agent.add_mcp_server(
    name="my_server",                         # 工具名前缀
    command=["python", "my_mcp_server.py"],   # 启动命令
    env={"API_KEY": "..."},                   # 可选环境变量
)
# 工具自动注入 DeferredToolRegistry，LLM 通过 tool_search 内置工具按需 promote
```

### 场景 7：多 Agent
```python
from neoagent.multi import Orchestrator, WorkerCard

orchestrator = Orchestrator(config, max_depth=2, max_concurrent_workers=5)
orchestrator.register_worker(WorkerCard(
    name="researcher",
    description="Searches and summarizes information",
    instruction="You are a research specialist...",
    tags=("research", "web"),
    tools=("bash", "grep"),
))
result = await orchestrator.run("Research topic X and write a report")
await orchestrator.close()
```

### 场景 8：HTTP 服务
```python
from neoagent.channels import FastAPIChannel

channel = FastAPIChannel(agent)
await channel.run(host="0.0.0.0", port=8000)
# POST /chat → {"message": "..."} → SSE stream
```

### 场景 9：断点续传
```python
from pathlib import Path
from neoagent.session import Session

config = NeoAgentConfig(..., session_dir=Path("~/.neoagent/sessions").expanduser())

session = await Session.resume(session_id="abc-123", storage=agent._storage)
result = await agent.chat("Continue from where we left off", session=session)
```

### 场景 10：可观测性
```python
from pathlib import Path
from neoagent.observe import Observer
from neoagent.observe_subscriber import ObserverSubscriber
from neoagent.eval import EvalRunner

# 终端彩色日志 + 文件输出
observer = Observer(console=True, log_dir=Path("logs/"))
subscriber = ObserverSubscriber(observer)
subscriber.attach(agent._event_bus)   # 订阅 agent 的 EventBus

# 关闭后清理
subscriber.detach(agent._event_bus)
observer.close()

# 评估质量
runner = EvalRunner(agent)
metrics = await runner.run(test_cases)
```

---

## 6. 深度参考

| 需要 | 去哪里 |
|------|--------|
| 完整 API 文档 | `/Users/neo/Desktop/project/git/neoagent/docs/guide.md` |
| 模块源码 | `neoagent/core/`, `neoagent/tools/`, `neoagent/multi/`, `neoagent/mcp/` |
| 架构原理（为什么这样设计） | `wiki/` 对应 12 页（通过 ai-knowledge skill 导航） |
| 安装 | `pip install -e /path/to/neoagent`（v0.1.0，未发布 PyPI） |

---

## 7. 已知 API 缺口（写代码时注意）

| 缺口 | 现状 | 影响 |
|------|------|------|
| Skills 注册 | `agent._prompt_builder`（私有属性） | 需通过私有属性访问，未来版本可能变更 |
| EventBus 订阅 | `agent._event_bus`（私有属性） | 同上 |
| Observability 接入 | `agent._event_bus`（私有属性） | ObserverSubscriber 需通过私有属性访问 EventBus |

---

## 8. 实现计划

### Phase 1：创建 Skill 文件
创建 `~/.claude/skills/neoagent/SKILL.md`，包含：
- frontmatter（name + description）
- 模块接线图
- 场景快查表
- 10 个代码骨架
- 深度参考

### Phase 2：验证
用一个实际任务验证：
- "帮我用 neoagent 创建一个带记忆的代码助手 agent"
- LLM 应能通过 skill 定位到 场景 1 + 场景 3，写出正确代码

### Phase 3：迭代
根据实际使用中发现的问题更新 skill 内容。
