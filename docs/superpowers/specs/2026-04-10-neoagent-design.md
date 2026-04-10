# neoagent — Python Agent Framework Design Spec

**日期**: 2026-04-10  
**状态**: 已确认（brainstorming 阶段产出）

---

## 1. 项目定位

- **名称**: neoagent（暂定）
- **语言**: Python
- **定位**: 内部自用，自主可控的 agent 框架
- **终态目标**: 完整框架覆盖 11 个组件维度
- **核心哲学**: Claude Code 骨架 + 各家优点 + 我们自己的设计
- **知识基础**: 基于 AI 工程知识库（5 个框架深度分析、11 个组件维度），每个组件选型有 KB 分析依据

---

## 2. 设计原则

- **设计层博采众长**: 每个组件从 KB 的 L1 页面分析中选最优设计思路
- **实现层以一家为主干**: 以 Claude Code 为骨架，吸收其他框架的具体改进点，保证内部一致性。不是 Frankenstein 式拼接
- **工具移植不重写**: 成熟开源项目的工具（特别是 Claude Code 的 Read/Write/Edit/Bash/Grep/Glob）直接移植适配，减少重复造轮子
- **生产级确定性**: 避免概率性方案在核心链路（如预测预加载），优先确定性和稳定性
- **分层递进**: v1 极简核 → v2 记忆+上下文 → v3 多 agent+hooks+MCP
- **YAGNI**: 不做的就不做，等真正痛了再加

---

## 3. 版本路线图

| 版本 | 包含模块 |
|------|---------|
| v1 极简核 | Query Loop + Tool System + Prompt System + 基础上下文压缩 |
| v2 实用核 | + Memory System + Context Management（完整版）+ 技能懒加载 |
| v3 完整框架 | + Multi-Agent + Hooks + MCP + Channel |

---

## 4. v1 模块布局

```
neoagent/
├── core/
│   ├── loop.py        # Query Loop — 简单 while 循环（KB 方案 A）
│   ├── prompt.py      # Prompt System — Section-based 动态组装（KB 方案 B）
│   ├── compress.py    # 基础上下文压缩
│   └── types.py       # 核心类型 Message / Turn / ToolCall / ToolResult
├── tools/
│   ├── base.py        # BaseTool 协议（Pydantic input model）
│   ├── registry.py    # ToolRegistry — 静态注册（KB 方案 A）
│   ├── permission.py  # PermissionChecker — auto/ask/deny 三级制
│   └── builtin/       # 移植的工具（Read/Write/Edit/Bash/Grep/Glob）
├── providers/
│   └── anthropic.py   # Claude API 适配（v1 只做一个 provider）
├── config.py          # 配置
└── agent.py           # 顶层 Agent 入口
```

---

## 5. 核心类型

```python
@dataclass
class Message:
    role: Literal["user", "assistant", "system", "tool_result"]
    content: str | list[ContentBlock]

@dataclass
class ToolCall:
    id: str
    name: str
    input: dict

@dataclass
class ToolResult:
    call_id: str
    output: str
    is_error: bool = False

@dataclass
class Turn:
    messages: list[Message]
    tool_calls: list[ToolCall]
    tool_results: list[ToolResult]
    stop_reason: Literal["end_turn", "tool_use", "max_tokens"]

@dataclass
class ConversationResult:
    turns: list[Turn]
    reason: Literal["completed", "max_turns"]
```

---

## 6. Query Loop 详细设计

**选型依据**: KB 方案 A（简单循环），编程助手/CLI 场景最优  
**骨架**: Claude Code（双核分工设计思路）  
**吸收改进**: DeerFlow 多条件触发 + Hermes tool pair 修复

### 核心 API

```python
class QueryLoop:
    def __init__(
        self,
        provider: Provider,
        tool_registry: ToolRegistry,
        prompt_builder: PromptBuilder,
        max_turns: int = 30,              # KB: 必须有硬上限
        context_budget: int = 0,          # 0 = 自动从 provider 获取
        on_turn: Callable | None = None,  # 轻量回调，可观测每轮
    ): ...

    async def run(self, messages: list[Message]) -> ConversationResult:
        for turn_idx in range(self.max_turns):
            # 0. 检查 token 预算，超限则压缩
            if self._estimate_tokens(messages) > self.context_budget * 0.7:
                messages = await self._compress(messages)

            # 1. 组装 prompt
            system = self.prompt_builder.build()

            # 2. 调模型
            response = await self.provider.create(
                system=system,
                messages=messages,
                tools=self.tool_registry.get_schemas(),
            )

            # 3. 无工具调用 → 结束
            if response.stop_reason == "end_turn":
                return ConversationResult(turns=turns, reason="completed")

            # 4. max_tokens 命中 → 降级重试（Claude Code 模式）
            if response.stop_reason == "max_tokens":
                response = await self._retry_with_lower_max(...)

            # 5. 执行工具（支持并行）
            results = await self._execute_tools(
                response.tool_calls,
                permission_checker=self.tool_registry.check_permission,
            )

            # 6. 结果追加到消息流，进入下一轮
            messages.extend(self._build_tool_messages(response, results))

        return ConversationResult(turns=turns, reason="max_turns")
```

### 关键设计点

- **max_turns 硬上限**: KB 反复强调不设上限就是烧钱
- **max_tokens 降级重试**: 来自 Claude Code，不直接失败
- **工具并行执行**: 来自 OpenHarness asyncio.gather()，加安全判定
- **on_turn 回调**: 轻量可观测，v1 不搞 Hermes 的 9 种回调
- **token 预算检查在循环开头**: 每轮开始前检查，不等溢出再救

---

## 7. 上下文压缩设计

**组合方案**: DeerFlow 的触发 + Claude Code 的压缩 + Hermes 的修复

### Token 估算

- 用 tiktoken 精确计数（不用 OpenHarness 的字符/4，中文误差太大）
- tools schema 纳入估算（Hermes 教训：50+ 工具额外 20-30K token）

### 触发机制

- DeerFlow 的阈值模式：占 context window 70% 时自动触发
- 自动触发，不靠手动检测

### 压缩策略

- v1：Claude Code 的 LLM 摘要替换（方案 B）
- v2：升级到 Hermes 的迭代摘要（方案 B2，带 `_previous_summary`）

### 熔断机制

- Claude Code 的连续失败熔断（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）
- 补上 Claude Code 没做的降级：熔断后截断最旧消息（而非什么都不做）

### 压缩后修复

- Hermes 的 `_sanitize_tool_pairs()`：修复孤儿 tool result/call
- 确保压缩输出是合法对话格式，能继续驱动工具调用

---

## 8. Tool System 详细设计

**骨架**: Claude Code（工具执行运行时、权限系统、并发安全）  
**吸收改进**: OpenHarness Pydantic 简洁性

### 核心 API

```python
# --- tools/base.py ---
class BaseTool(ABC):
    name: str
    description: str                    # 20-50 字，KB: 超 200 字浪费 token
    input_model: type[BaseModel]        # OpenHarness: Pydantic 一套定义两用
    permission: Literal["auto", "ask", "deny"] = "ask"  # Claude Code: 三级权限
    is_concurrent_safe: bool = False    # Claude Code: fail-closed 默认串行

    @abstractmethod
    async def execute(self, input: BaseModel) -> ToolResult: ...

    def get_schema(self) -> dict:
        """自动从 input_model 生成 API schema"""
        return self.input_model.model_json_schema()

# --- tools/registry.py ---
class ToolRegistry:
    def register(self, tool: BaseTool): ...
    def get_schemas(self) -> list[dict]:
        """deny 的不暴露给模型"""
        return [t.get_schema() for t in self._tools.values()
                if t.permission != "deny"]

    async def execute(self, calls: list[ToolCall]) -> list[ToolResult]:
        """并行/串行执行 — Claude Code 并发安全判定"""
        safe, unsafe = self._partition_by_concurrency(calls)
        results = await asyncio.gather(*[self._run(c) for c in safe])
        for c in unsafe:
            results.append(await self._run(c))
        return results

# --- tools/permission.py ---
class PermissionChecker:
    async def check(self, tool: BaseTool, input: BaseModel) -> bool:
        if tool.permission == "auto": return True
        if tool.permission == "deny": return False
        return await self._ask_user(tool, input)
```

### 工具移植清单（v1）

从 Claude Code 移植，适配 BaseTool 接口：

| 工具 | 类别 |
|------|------|
| Read | 文件操作 |
| Write | 文件操作 |
| Edit | 文件操作 |
| Glob | 文件操作 |
| Grep | 搜索 |
| Bash | 执行 |
| Agent | 子代理（可选，v1 可后置） |

### 关键设计点

- Pydantic input_model 一套两用（验证 + schema）— OpenHarness 模式
- auto/ask/deny 三级权限 — KB: 生产环境最低门槛
- fail-closed 并发（默认串行）— 安全优先
- 工具结果截断防 context 溢出 — Claude Code + Hermes

---

## 9. Prompt System 详细设计

**骨架**: Claude Code（Section-based 动态组装 + 静态/动态边界）  
**吸收改进**: DeerFlow 技能懒加载（v2）

### 核心 API

```python
class PromptSection:
    name: str
    content: str | Callable[[], str]  # 静态字符串 or 动态生成函数
    priority: int                      # 越小越优先（放在 prompt 前部）
    is_static: bool = True             # 静态/动态标记 — Claude Code 缓存边界

class PromptBuilder:
    def __init__(self):
        self._sections: list[PromptSection] = []

    def add_section(self, section: PromptSection): ...

    def build(self) -> str:
        """按优先级排序，静态在前动态在后"""
        sorted_sections = sorted(self._sections,
            key=lambda s: (not s.is_static, s.priority))
        parts = []
        for s in sorted_sections:
            content = s.content if isinstance(s.content, str) else s.content()
            parts.append(f"# {s.name}\n{content}")
        return "\n\n".join(parts)
```

### 预置 Sections

```python
# 静态（缓存命中）
PromptSection("identity", "你是 neoagent...", priority=0, is_static=True)
PromptSection("rules", load_rules_md(), priority=1, is_static=True)
PromptSection("failure_modes", FAILURE_SUPPRESSION, priority=2, is_static=True)

# 动态（每轮重建）
PromptSection("environment", lambda: get_env_info(), priority=10, is_static=False)
PromptSection("project_context", lambda: load_claude_md(), priority=11, is_static=False)
```

### 关键设计点

- 静态/动态边界分离 — Claude Code 缓存优化
- 失败模式前置 — KB: Lost-in-the-middle 对策
- Section priority 显式排序 — 比 DeerFlow 列表下标更可控（我们的改进）
- Callable 动态生成 — 比 Hermes 9 层流水线简洁
- v1 全量暴露工具，v2 加 DeerFlow 式懒加载（确定性按需加载）
- 预测预加载已删除 — 生产系统要确定性

---

## 10. Provider 层

v1 只做 Anthropic Claude API 适配，薄封装：

```python
class Provider(ABC):
    @abstractmethod
    async def create(self, system: str, messages: list, tools: list) -> Response: ...

class AnthropicProvider(Provider):
    def __init__(self, api_key: str, model: str = "claude-sonnet-4-20250514"):
        self.client = AsyncAnthropic(api_key=api_key)
        self.model = model

    async def create(self, system, messages, tools) -> Response:
        # 封装 anthropic SDK 调用
        ...
```

---

## 11. 各组件设计来源总览

| 组件 | 骨架 | 关键吸收 | 自研改进 |
|------|------|---------|---------|
| Query Loop | Claude Code | DeerFlow 触发 + Hermes 修复 | 熔断降级补完 |
| Tool System | Claude Code | OpenHarness Pydantic | 工具直接移植 |
| Prompt System | Claude Code | DeerFlow 懒加载(v2) | Section priority 显式排序 |
| 上下文压缩 | Claude Code | DeerFlow 多条件触发 + Hermes tool pair 修复 | 熔断后截断降级 |

---

## 12. 非目标（v1 明确不做）

- Multi-Agent 多 agent 协作
- Hooks 事件驱动扩展
- MCP 协议支持
- Memory System 跨会话记忆
- 技能懒加载
- 预测预加载（所有版本都不做）
- 多 Provider 支持（OpenAI/Gemini）
- Web UI / Dashboard
