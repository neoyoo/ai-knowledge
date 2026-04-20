---
title: LLM-Driven Context Lifecycle
aliases: [上下文生命周期, freed-recall protocol, auto_free_after (deprecated)]
kind: pattern
created: 2026-04-19
updated: 2026-04-20
concepts_involved: [[tool-system]], [[context-management]], [[prompt-system]], [[hooks]]
reference_implementations: [neoagent (post-2026-04-20)]
status: mature
supersedes: tool-metadata-driven-context-lifecycle (v1, 2026-04-19)
---

## 一句话定义

Agent 运行时通过 LLM + prompt 规则主动管理 tool_result 和历史消息的生命周期，由多层互补防线替代单一自动机制。

## 触发问题

Agent 在对话循环中调用工具（`run_python`、`fetch_html`、`read_large_file` 等），**工具产出的 `tool_result` 体积可能很大**（几 KB 到几十 KB）。三种传统应对都有缺点：

| 做法 | 代价 |
|------|------|
| 全部保留在 history | 几轮下来就撑爆 context，query_loop 被迫提前压缩 |
| 调用完就丢弃 | LLM 后续想再引用就没了（需要重新调用工具，浪费） |
| 交给全局压缩/截断 | 粒度是"整条 message"，无法针对"某个 tool 调用"做差异化决策；压缩后信息失真 |

真实需求：**大部分 tool_result 在 LLM 消化过一次后就不再需要详细内容，但少数情况可能要召回完整内容。需要粒度到"每个 tool 调用"的生命周期控制。**

## 参与的概念

这个模式横跨四个 L1 概念，不归任何一个单独管：

- [[tool-system]] — 提供 `free_tool_result` / `recall_tool_result` 两个 LLM 可调元工具
- [[context-management]] — 全局 ContextCompressor 作为安全网；消息重写逻辑
- [[prompt-system]] — MANDATORY_CONTEXT_RULES 硬规则 + WORKING_MEMORY 动态注入
- [[hooks]] / events — 观测 free / recall 行为（审计、日志）

## 核心设计：四层互补防线

### Layer 1 — MANDATORY_CONTEXT_RULES（prompt 硬规则）

在 system prompt 的 `<MANDATORY_CONTEXT_RULES>` 块里显式要求 LLM：在工具产出大量内容之后，下一个 tool_use 必须是 `free_tool_result`（除非内容仍被后续步骤直接需要）。

全大写标签名本身即具有强制语义信号——来自 neoagent pattern 调试教训：早期用自然语言指引 LLM 折叠工具结果，遵守率很低；改用 `<MANDATORY_CONTEXT_RULES>` 标签后行为即变（参考 [[prompt-system]] 中"通用执行纪律对特定模型失效"的陷阱）。

trip-os subagent 的实际规则文本（来自 `docs/context-layout-spec.md` Layer 3 示例）：

```xml
<MANDATORY_CONTEXT_RULES>
1. 工具结果折叠规则
   - 当 working_memory 中出现 <freed_tool_results> 清单时，说明部分工具结果已折叠
   - 若需要查看某个折叠结果的完整内容，调用 recall_tool_result(tool_use_id)
   - 不要对折叠的内容进行猜测或填充
</MANDATORY_CONTEXT_RULES>
```

Layer 1 解决"LLM 主动忘记"的第一道门——当 LLM 遵守规则时，不需要其他层介入。

### Layer 2 — WORKING_MEMORY 7 段迭代摘要

Goal / Progress / Decisions / Files / Next Steps / Open Questions / Constraints——把被 free 的工具产出蒸馏到结构化状态里。即使原内容折叠后，LLM 仍然能看到 Goal 和 Decisions，不会"忘了自己在做什么"。

`<working_memory>` 追加到 system prompt 的 ephemeral 区（`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 之后），每轮重建，不影响 prefix cache（参考 Hermes Agent 的冻结前缀 + ephemeral 追加模式，[[prompt-system]] 方案 C）。

关键子标签：`<freed_tool_results>` — 在 WORKING_MEMORY 里维护已折叠工具结果的可见清单，让 LLM 知道"这些信息还在，只是折叠了，需要时我能拿回来"：

```
<freed_tool_results>
  - toolu_0xAB12 · fetch_and_clean · 18.3KB · '<h1>Arashiyama Bamboo Grove...'
  - toulu_0xCD34 · extract_structured · 2.1KB · '{"attractions": [{"name": "Bamboo...'
</freed_tool_results>
```

**迭代摘要机制**（来自 Hermes Agent `_previous_summary`，`context_compressor.py:2635`）：每次压缩时把前次摘要作为前置上下文，LLM 只需"更新 Progress、继承 Goal / Decisions"——解决"越压越忘"问题。GOAL 和 KEY_DECISIONS 跨多次压缩自动保留。

Layer 2 解决"被 free 的信息不能让 LLM 完全遗忘状态"的问题——即使全部工具结果都折叠了，LLM 仍知道整体进度。

### Layer 3 — ContextCompressor 全局压缩（触发式安全网）

三模式触发（参考 DeerFlow `SummarizationMiddleware`）：`token_count`（建议 70% 窗口阈值）/ `message_count` / `fraction`，任意满足其一触发。

触发后：旧消息打 `COMPRESSED` 标记而非物理删除（参考 AgentScope `update_messages_mark(COMPRESSED)`），正常检索路径跳过，调试/审计时按 ID 可查。压缩失败时保留原始消息（参考 Claude Code 熔断策略，反对 Hermes 的静默删除冷却期）。

Layer 3 是第二道安全网——当 LLM 没有足够及时地调用 `free_tool_result` 时，ContextCompressor 在阈值处兜底，确保 context 不会无限膨胀。

### Layer 4 — LLM 可见的两个元工具

- `free_tool_result(tool_use_ids: list[str])` — LLM 主动折叠，将 tool_result 替换为占位符（`[freed: id=..., preview=...]`），原内容保留在 session state
- `recall_tool_result(tool_use_id: str)` — 拿回折叠内容，**仅本轮可见**，下轮自动回到折叠态

消息重写是**纯视图层操作**——在每次 provider call 前非破坏性重写一个 messages 副本，真实 session state 始终保留原内容。折叠可逆、可审计、不丢数据。

## 为什么四层互补

| 层 | 解决的问题 | 不能单独 work 的原因 |
|----|-----------|---------------------|
| Layer 1 MANDATORY 规则 | LLM 主动、及时 free | LLM 有时会忽略规则，或在上下文超长后遵守率下降 |
| Layer 2 WORKING_MEMORY | 折叠后任务状态不丢失 | 不能代替 free 本身；没有 Layer 1/4，内容不会被折叠 |
| Layer 3 ContextCompressor | context 超限时的兜底 | 粒度是整条消息，无法针对单个 tool_result 精细控制 |
| Layer 4 元工具 | LLM 主动 free + 按需 recall | 仅靠 LLM 自觉调工具会漏；没有 recall，折叠就变成"永久丢失" |

## 设计演进（v1 → v2）

### v1（2026-04-19）的方案：`auto_free_after` + MANDATORY 规则

原始思路：工具基类增加 `auto_free_after: int` 字段，框架在 N 轮后自动 free。配合 MANDATORY 规则，形成"框架自动 + LLM 主动"两道防线。

v1 在没有 WORKING_MEMORY 迭代摘要（Layer 2）的情况下是合理的起点——`auto_free_after` 在 MANDATORY 规则仍不够稳定时补的安全网，逻辑上自洽。

### 为什么 v1 被淘汰

**1. 自循环 bug（决定性证据）**

`recall_tool_result` 设了 `auto_free_after = 1`（作者原意是"防止永久 re-inflate"）。结果：LLM 每次 recall 回来的内容下一轮又被框架自动 free，LLM 不得不再 recall 一次……

**trip-os 2026-04-20 E2E 跑 test/1.jpg，subagent 连续 5 次 recall 同一内容**，每次都触发 auto_free（log 路径 `.runs/135aff4b-a578-4f76-a9cc-ea80de440f37/logs/2026-04-20-21-18-54-sub.log` 第 162-220 行）。`auto_free_after` 和 MANDATORY 规则本应互补，实际在 recall 这条路径上**互相打架**。

**2. 机制/策略混淆**

`auto_free_after` 把"何时 free"的策略焊死在工具元数据上。但工具作者**不知道**当前任务需要这个输出多久——那是 prompt / LLM 层面的上下文知识。`run_python` 的作者写 `auto_free_after=2`，但如果当前任务是一个需要多次对比同一段输出的分析任务，2 轮后 auto free 就是过早折叠。

**3. 四层架构下变为冗余**

v2 的 Layer 1 + 2 + 3 已经覆盖了 `auto_free_after` 的全部原始安全网需求：
- Layer 1（MANDATORY 规则）处理"LLM 应该 free 的时机"
- Layer 3（ContextCompressor）处理"框架兜底压缩"
- 中间不再需要一个"按工具固定轮次自动触发"的机制

### v2（2026-04-20）的方案

完全移除 `auto_free_after` 字段，纯 LLM-driven，四层互补。

同步动作：
- neoagent SDK 移除 `BaseTool.auto_free_after` 字段
- `neoagent/core/loop.py` age 追踪逻辑和 auto-free 触发已删除
- `neoagent/session.py` `tool_result_ages` 字段已删除
- trip-os 已删除工具的 `auto_free_after` override
- `docs/context-layout-spec.md` 第 12.2 节加了反决策条目

**反思**：v1 不是"错误的设计"，是"在 WORKING_MEMORY 层（Layer 2）还不存在时的合理补丁"。v2 淘汰它是因为架构演进后 auto 层变冗余，且在 recall 路径上引发了难以预测的副作用。

## 参考实现

**neoagent**（post-2026-04-20，`/Users/neo/Desktop/project/git/neoagent/`）

关键代码路径（v2 版本）：

- `tools/builtin/free_tool_result.py` — LLM 主动折叠
- `tools/builtin/recall_tool_result.py` — 本轮回溯（已移除 `auto_free_after=1`）
- `session.py` / `FreedToolResult` — 存储（已移除 `tool_result_ages` 字段）
- `core/loop.py` / `_apply_freed_to_messages()` — 非破坏性消息重写
- `core/loop.py` / `_render_freed_section()` — WORKING_MEMORY 的 `<freed_tool_results>` 清单生成
- `events.py` / `ToolResultFreedEvent` / `ToolResultRecalledEvent` — 可观测

注：`BaseTool.auto_free_after` 字段已移除，`core/loop.py` 的 age 追踪逻辑已删除。

## 何时不该用

- **场景本身不会触及 context 上限** — 折叠协议的引入成本大于收益
- **单轮对话为主** — 没有"后续可能 recall"的需求
- **LLM 模型不够强** — 较小模型可能不懂占位符含义或忘记调 recall，Layer 1 规则遵守率低

## 和相邻 pattern 的关系

- 和 [[context-management]] 的传统压缩**互补**，不替代——压缩处理"对话历史"，折叠处理"tool 产出"
- Layer 2 的 WORKING_MEMORY 和 [[prompt-system]] 里的"冻结前缀 + ephemeral 追加"方案紧密相关——WORKING_MEMORY 正是 ephemeral 层的主要内容之一
- 如果和 MCP 工具池的 [[mcp-skills#deferred tool registry]] 组合，可以形成"工具可见性生命周期 + 工具结果生命周期"的对称设计

## 溯源

- v1 初始设计：trip-os 项目 2026-04-17 到 2026-04-19 的若干次 agent 运行
- MANDATORY 标签必要性：早期 skill 里温和指导 LLM 折叠，结果 LLM 从来不主动调 free；加了标签规则后行为即变
- v1 → v2 演进契机：2026-04-20 跑 test/1.jpg 时抓到 recall 自循环（log 路径 `.runs/135aff4b-a578-4f76-a9cc-ea80de440f37/logs/2026-04-20-21-18-54-sub.log` 第 162-220 行）
- v2 架构整合：对比了 AgentScope（COMPRESSED 标记）+ Hermes Agent（7 段迭代摘要）+ Claude Code（SYSTEM_PROMPT_DYNAMIC_BOUNDARY）后形成——见 `docs/context-layout-spec.md` 第 12.2 节反决策表
