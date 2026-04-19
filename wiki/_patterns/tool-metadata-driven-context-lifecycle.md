---
title: Tool-Metadata-Driven Context Lifecycle
aliases: [工具元数据驱动的上下文生命周期, freed-recall protocol, auto_free_after]
kind: pattern
created: 2026-04-19
concepts_involved: [[tool-system]], [[context-management]], [[prompt-system]], [[hooks]]
reference_implementations: [neoagent]
status: mature
---

## 触发问题

Agent 在对话循环中调用工具（`run_python`、`fetch_html`、`read_large_file` 等），**工具产出的 `tool_result` 体积可能很大**（几 KB 到几十 KB）。三种传统应对都有缺点：

| 做法 | 代价 |
|------|------|
| 全部保留在 history | 几轮下来就撑爆 context，query_loop 被迫提前压缩 |
| 调用完就丢弃 | LLM 后续想再引用就没了（需要重新调用工具，浪费） |
| 交给全局压缩/截断（传统 context-management） | 粒度是"整条 message"，无法针对"某个 tool 调用"做差异化决策；压缩后信息失真 |

真实需求：**大部分 tool_result 在 LLM 消化过一次后就不再需要详细内容，但少数情况可能要召回完整内容。需要粒度到"每个 tool 调用"的生命周期控制**。

## 参与的概念

这个模式**横跨四个 L1 概念**，不归任何一个单独管：

- [[tool-system]] —— 工具需要自声明"我的输出多少轮后可以折叠"（协议字段）
- [[context-management]] —— 在 query loop 里按字段扫描并执行折叠
- [[prompt-system]] —— 动态往 system prompt 注入"可恢复清单"
- [[hooks]] / events —— 观测折叠行为（审计、日志）

## 协议

### 层 1 — 工具元数据协议字段

工具基类增加一个字段：

```
class BaseTool:
    auto_free_after: int = 0     # 0 表示不自动折叠；N 表示 N 轮后折叠
```

这是**协议**：工具自己知道"我输出的东西通常什么时候就失去信息价值了"，把决策权下放到工具实现方。

| 工具类型 | 建议值 | 理由 |
|---------|--------|------|
| `run_python` / `read_file` 返大数据 | 1–3 | 用完就该折叠，正常情况不需要回看 |
| `search_web` / `list_dir` | 5–10 | 列表类结果可能多次决策复用 |
| `user_ask` / `log_write` | 0 | 小输出不折叠 |

### 层 2 — 生命周期状态机

每个 tool_use_id 对应一个 `ToolResult` 对象，维护 4 个字段：

```
ToolResult:
    id: str
    tool_name: str
    age: int              # 自产出以来过了多少轮
    status: "live" | "freed"
    original: str         # 原内容（无论 freed 与否都保留在 session state）
    placeholder: str      # 折叠后替换进 message 的占位符
```

状态转移：

```
live  --(age >= tool.auto_free_after)-->  freed
live  --(LLM 调用 free_tool_result)-->    freed
freed --(LLM 调用 recall_tool_result)-->  live (仅本轮可见)
live  --(下一轮)-->                       freed (回到折叠态)
```

### 层 3 — message 重写（不破坏 history）

在每次 provider call 前，**非破坏性**重写一个 messages 副本：

```
for msg in messages:
    for block in msg.content:
        if block is ToolResultBlock and block.tool_use_id in freed_ids:
            replace content with placeholder: "[freed: id=..., preview=...]"
```

真实 session state 保留原内容，LLM 看到的是占位。这样折叠是**纯视图层的事**，可恢复、可审计、不丢数据。

### 层 4 — system prompt 动态注入

在构建 system prompt 时追加一段：

```
## Freed Tool Results (recoverable)

- `toolu_abc` · run_python · 3.5KB · 'print("total rows", 1234)...'
- `toolu_def` · fetch_html · 12KB · '<html><head>...'

Call recall_tool_result(tool_use_id) to re-materialize for this turn.
```

LLM 就知道"**这些信息还在，只是折叠了，需要时我能拿回来**"。

### 层 5 — 两个 LLM 可见的元工具

- `free_tool_result(tool_use_ids: list[str])` — 手动折叠
- `recall_tool_result(tool_use_id: str)` — 拿回原内容，仅本轮可见

LLM 大部分时候不用调（auto_free_after 自动干活），但在"我主动判断这个工具结果不再需要"时可以主动调 free，或者被动恢复时调 recall。

### 层 6 — 事件观测（可选但推荐）

```
ToolResultFreedEvent(tool_use_id, tool_name, size, preview, reason)
ToolResultRecalledEvent(tool_use_id, tool_name)
```

EventBus 广播，Observer 日志里打 `[FREED] toolu_abc · run_python · 3.5KB · 'print("total...'`。

**为什么重要**：没有这层，LLM 行为是黑盒——你不知道 auto_free_after 什么时候被触发、LLM 为什么要 recall。一上线就瞎。

## 为什么这样设计

| 设计点 | 为什么 | 反思 |
|-------|-------|------|
| 把"老化时间"放在工具元数据 | 工具作者最清楚自己输出的语义寿命 | 比"LLM 自己决定何时 free"可靠——LLM 常常忘 |
| message 重写而非修改 history | 保留原 history 的完整性，折叠可逆 | 付出一点内存代价换可逆性 |
| placeholder 带 preview | LLM 能判断"这个折叠的东西是不是我要的" | 没 preview 的话 recall 就变成盲摸 |
| system prompt 动态清单 | 让 LLM"感知"有哪些可恢复内容 | 不注入，LLM 会忘记折叠的存在 |
| 双触发：auto + manual | 不信任任一方单独 work | **关键教训**：只靠 LLM 自觉 free 会漏；只靠 auto 会过早折叠真正需要的。两条都要有 |
| MANDATORY 标签规则 | LLM 看到非标签指令容易忽略 | **关键教训**：直接写 `<MANDATORY_CONTEXT_RULES priority="critical">` 的标签，遵守率显著提升 |
| 事件观测 | 否则全是黑盒 | 工程必需，不是可选 |

## 参考实现

**neoagent** (`/Users/neo/Desktop/project/git/neoagent/`，shelved but designed this pattern)

关键文件路径（完整协议跨 5 个文件）：
- `neoagent/tools/base.py` — `BaseTool.auto_free_after` 字段定义
- `neoagent/session.py` — `FreedToolResult` 数据类、session state 存储
- `neoagent/core/loop.py` — `_apply_freed_to_messages()` 非破坏重写、age 递增、auto-free 触发
- `neoagent/core/loop.py` — `_render_freed_section()` 生成 system prompt 注入段
- `neoagent/tools/builtin/free_tool_result.py` / `recall_tool_result.py` — LLM 可见的元工具
- `neoagent/events.py` — `ToolResultFreedEvent` / `ToolResultRecalledEvent` 事件

## 何时不该用

- **工具输出都很小**（每条 <1KB） — 折叠协议的开销比收益大
- **单轮对话为主** — 没有"后续可能 recall"的需求
- **已经上了足够激进的全局压缩** — 折叠是更温和的方案，和全局压缩并存可能造成决策二义性
- **LLM 模型不够强** — 较小模型可能不懂占位符含义或忘记调 recall

## 和相邻 pattern 的关系

- 和 [[context-management]] 的传统压缩**互补**，不替代——压缩处理"对话历史"，折叠处理"tool 产出"
- 如果和 MCP 工具池的 [[mcp-skills#deferred tool registry]] 组合，可以形成"**工具可见性生命周期 + 工具结果生命周期**"的对称设计

## 溯源

- 本 pattern 的设计讨论：trip-os 项目 2026-04-17 到 2026-04-19 的若干次 agent 运行
- MANDATORY 标签的必要性来自一次失败调试：早期 skill 里温和指导 LLM 折叠，结果 LLM 从来不主动调 free——加了标签规则后行为即变
- auto_free_after 是在 MANDATORY 标签仍不够稳定后补的"安全网"
