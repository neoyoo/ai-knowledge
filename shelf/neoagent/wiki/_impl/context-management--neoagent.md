---
title: "context-management — neoagent"
category: L2
parent: "[[context-management]]"
source: neoagent
source_version: "9d3621d3cf2717ee989dd954bd159d9d4c9a20db"
concept: context-management
confidence: high
created: 2026-04-19
updated: 2026-04-19
---

## 概述

neoagent 的上下文管理采取**双层主动策略**：一层是经典的 `ContextCompressor`（tiktoken 精确计数、70% 阈值触发、结构化摘要、前次摘要延续、3 次失败后降级截断）；另一层是**开创性的 freed/recall 机制**——工具协议层 `BaseTool.auto_free_after` 字段让单个 tool_result 到期后自动替换为 placeholder（size + 80 字预览），同时 `recall_tool_result` 工具让 LLM 主动从 freed 状态恢复单条内容"仅当前轮可见"。这一设计把"压缩整个历史"升级为"按工具粒度细粒度生命周期管理"，是同类项目中独一无二的形式化上下文回收协议。

## 架构分析

### 两层上下文管理的职责划分

| 层次 | 触发方式 | 粒度 | 可逆 |
|------|---------|------|------|
| `ContextCompressor` | token 总量 > 70% 预算 | 整段对话被摘要替换 | 通过 `previous_summary` 迭代保留，不可回滚 |
| `auto_free_after` + freed/recall | 单个 tool_result 到 age 阈值 | 单条 ToolResultBlock | 完全可逆——`original_content` 保留在 session state，可随时 recall |

两层独立工作且叠加生效：被 freed 的 tool_result 在 provider 视图里是 placeholder，tiktoken 计数自然变小；ContextCompressor 要压缩整体时，看到的是已 freed 的精简视图，进一步摘要。

### ContextCompressor 实现细节

- **估算**：`estimate_tokens(msgs)` 把每条 message 转文本（tool_use 序列化为 `[tool_use id=... name=... input=...]`、tool_result 转 `[tool_result id=... content=...]`），用 `tiktoken.get_encoding("cl100k_base")` 精确 encode。`estimate_tools_tokens(schemas)` 对 json.dumps 的整个 schemas list encode——意味着 **tools schema 纳入 budget**（避免长 tool schema 静默吃 context）。
- **触发**：`should_compress = (msg_tokens + tool_tokens) > budget * 0.7`
- **摘要 prompt**（`_build_summarize_system`）：结构化 5 字段模板：GOAL / PROGRESS / DECISIONS / FILES / NEXT STEPS / KEY CONTEXT，前次摘要作为 "Previous summary (update incrementally — do not discard)" 前缀，让多轮压缩保留任务状态不清零。
- **保留策略**：anchor（第一条）+ [LLM 生成的 summary user message] + 最近 `_KEEP_RECENT=6` 条。
- **失败降级**：LLM 调用抛异常时 `compression_failures += 1`；达到 `max_failures=3` 后永久走 `_truncate_oldest`。
- **`_sanitize_tool_pairs`**：压缩/截断后扫描剩余 messages，把没配对的 `ToolUseBlock`（use_id 无对应 result）或 `ToolResultBlock`（tool_use_id 无对应 use）剔除，然后补"(context removed during compression)" placeholder 保证 role 交替——严防 provider 400 error。

### Freed / Recall 协议

核心数据结构在 `session.py`：

```python
@dataclass
class FreedToolResult:
    id: str                   # tool_use_id（原 ToolUseBlock.id）
    tool_name: str            # 从 session.state.tool_use_to_tool_name 查
    size: int                 # len(original_content.encode())
    preview: str              # 80-char preview with "…" ellipsis
    original_content: str     # 完整原文（持久化到 session JSON）

@dataclass
class SessionState:
    freed_tool_results: dict[str, FreedToolResult]    # 所有 freed 的工具结果
    recalled_this_turn: set[str]                       # 本轮已 recall 的 id（跳过 placeholder 重写）
    tool_use_to_tool_name: dict[str, str]              # id -> tool name 映射
    tool_result_ages: dict[str, int]                   # id -> 经过了几轮 tool_use
```

三个触发路径写入 `freed_tool_results`：

1. **Auto_free_after**：`QueryLoop.run()` 每轮 tool_use 结束后（`core/loop.py:296-337`），对所有 `ToolResultBlock.tool_use_id` age +1；超过对应 tool 的 `auto_free_after` 且尚未 freed，从 messages 找 `original_content` → 加入 freed_tool_results → 发射 `ToolResultFreedEvent(reason="auto_free_after")`。
2. **主动 `free_tool_result`**：LLM 调工具 `free_tool_result(tool_use_ids=[...])`（`tools/builtin/free_tool_result.py`）→ 从 ContextVar 取 session → 遍历 messages 抓 original_content → 同样写入 freed_tool_results。
3. **MCP 工具自动（如 `recall_tool_result` 自身）**：该工具声明 `auto_free_after=1`，下一轮自动 free——防止 recall 内容永久 re-inflate。

读取路径在 `QueryLoop.run()` 开头：

```python
if session_state.freed_tool_results:
    msgs_for_llm = _apply_freed_to_messages(msgs, freed, recalled_this_turn)
    freed_section = _render_freed_section(freed)
    system = system + "\n\n" + freed_section
```

`_apply_freed_to_messages` 返回消息列表的副本，把 freed 且未 recalled 的 `ToolResultBlock.content` 替换为 `[freed: tool_use_id=..., tool=..., size=NNN B, preview='...']`；原 msgs 保持完整。`_render_freed_section` 生成 system 段：

```
## Freed Tool Results (recoverable)
- `toolu_01A...` · run_python · 4321B · 'First 80 chars of output…'

Call `recall_tool_result(tool_use_id)` to view full content for the current turn.
```

### Recall 的一轮回溯

LLM 调 `recall_tool_result(tool_use_id="toolu_01A...")`：
1. `RecallToolResultTool.execute()` 从 ContextVar 取 session
2. 查 `session.state.freed_tool_results[id]` 取 `original_content`
3. 写入 `session.state.recalled_this_turn.add(id)`
4. 返回 original_content 作为 ToolResult

下一轮 `_apply_freed_to_messages` 判断 `tool_use_id in recalled_this_turn` 时跳过重写——**但 recall 工具自己的输出在下一轮之后会被 auto_free_after=1 自动 free**，所以 recall 内容"只在调用之后的一个完整 turn 期间对 LLM 可见"，永久膨胀被防住。

`QueryLoop` 在 end_turn 和 tool_use 两个分支结束时都调 `session_state.recalled_this_turn.clear()`——recalled 状态不跨 turn 延续。

### 关键代码路径

- `neoagent/core/compress.py:47-124` — ContextCompressor 全部流程
- `neoagent/core/compress.py:14-32` — 结构化摘要 prompt 模板
- `neoagent/core/compress.py:126-158` — `_sanitize_tool_pairs` 配对清理
- `neoagent/tools/base.py:14` — `auto_free_after: int = 0`
- `neoagent/session.py:14-40` — `FreedToolResult` + `SessionState`
- `neoagent/core/loop.py:285-337` — age 追踪 + auto_free_after 触发
- `neoagent/core/loop.py:347-397` — `_apply_freed_to_messages` + `_render_freed_section`
- `neoagent/tools/builtin/free_tool_result.py:32-80` — 主动 free
- `neoagent/tools/builtin/recall_tool_result.py:14-41` — recall（含 `auto_free_after=1`）
- `neoagent/events.py:111-124` — `ToolResultFreedEvent` / `ToolResultRecalledEvent`

## 设计亮点

- **工具协议层声明 `auto_free_after`**：`BaseTool.auto_free_after: int = 0` 把上下文生命周期从 QueryLoop 的判断逻辑降到工具开发者的责任范畴——run_python 作者知道自己产出一般短命（输出用一次就行），声明 2；recall 声明 1。这是把"何时忘记"做成 protocol contract 的原创设计。
- **freed placeholder 保留结构化元数据**：`[freed: tool_use_id=..., tool=..., size=NNN B, preview='...']` 让 LLM 知道"这条曾存在、多大、大概讲什么"，相比"整段摘要替换"信息更精准、选择 recall 更有把握。
- **provider 视图 vs session 真相完全分离**：`_apply_freed_to_messages` 返回副本而非 mutate，session 持久化的 messages 保留所有原文；这意味着 session resume 后 LLM 仍可 recall，不会因为 freed 导致数据丢失。
- **recall 输出自带 `auto_free_after=1`**：解决"recall 后下次压缩又把同一段巨量 content 重新送入 provider"的 re-inflation 问题——形成 **free → recall → 用一次 → auto-free → 可再 recall** 的可重入循环，不会永久污染 context。
- **前次摘要延续**：`_build_summarize_system(previous_summary)` 把上次摘要作为 "do not discard" 前缀，多次压缩后 GOAL / KEY CONTEXT 不清零——和 Hermes Agent 的迭代摘要策略一致。
- **压缩失败 3 次后永久降级截断**：`compression_failures >= max_failures` 后跳 `_truncate_oldest`，避免每轮都尝试 LLM 摘要拖慢——但保留了前 N 次的原始错误率用于观测。

## 局限性

- **Freed 机制内存常驻**：`freed_tool_results` 字典没有上限，长 session 中反复 auto_free 可能累积到 MB 级（所有 tool result 原文都在 session JSON 里）——应有 LRU 淘汰或按总 size 限制的策略。
- **`tool_result_ages` 和 `tool_use_to_tool_name` 无清理**：ContextCompressor 压缩掉旧 tool_use/result 对后，对应的 id 键变成僵尸残留，长 session 字典无界增长。
- **压缩不感知 freed 语义**：当 freed 占位文本 "[freed: ...]" 进入压缩时被当作普通文本摘要掉，freed 的"可 recall"语义在摘要后丢失——用户再问"刚才那个工具结果是什么"，agent 可能无法恢复。
- **压缩失败冷却不支持恢复**：达到 3 次失败后永久走截断，无"一段时间后重新尝试 LLM 摘要"机制——如果是一次网络抖动导致的失败，后果是永久降级。
- **`should_compress` 阈值 70% 硬编码**：只能通过改 compressor 源码或重写 `should_compress` 子类化调整，不支持从 `NeoAgentConfig` 配置。
- **无图片 / 多模态 token 估算**：tiktoken 只处理文本；如果工具返回图片（将来 Claude 支持直接返回图片 block），估算不准。
- **`_KEEP_RECENT=6` 是硬编码**：压缩后只保留最近 6 条——对工具密集场景（一轮 4 个 tool_use + 4 个 tool_result = 8 条 message），`6` 甚至不足以保留最后一个完整 turn。
- **`_sanitize_tool_pairs` 的 placeholder 插入可能污染**："(context removed during compression)" 作为 user/assistant 角色交替占位——LLM 看到会困惑。

## 来源

- 源码版本：9d3621d3cf2717ee989dd954bd159d9d4c9a20db
- 分析深度：源码级
- 关键文件：`neoagent/core/compress.py`, `neoagent/core/loop.py:159-169, 285-397`, `neoagent/session.py:14-40`, `neoagent/tools/builtin/{free,recall}_tool_result.py`, `neoagent/tools/base.py:14`
