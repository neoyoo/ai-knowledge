---
title: "MMD 高密度符号上下文图"
category: insight
source: tencentdb-agent-memory
concept: context-management
created: 2026-06-09
updated: 2026-06-09
tags: [context-management, memory-system, tool-system, symbolic-map, progressive-recall]
relations:
  - target: "[[memory-system--tencentdb-agent-memory]]"
    type: extracted_from
  - target: "[[symbolic-context-map-progressive-recall]]"
    type: informs
---

## 核心洞察

TencentDB Agent Memory 把大工具结果压缩成一张 **LLM 可读的 Mermaid/MMD 认知状态机**，让模型用图中的节点、状态和边关系维持任务方向感，再按需读取原文证据。

这不是普通摘要。它要求 L2 LLM 把多个 `tool_call_id` 映射到有限的 `node_id`，节点带 `status`、`summary`、`timestamp`，并允许用形状、连线、blocked 节点表达语义。效果是：上下文里保留的是**高密度任务地图**，不是流水账。

## 设计方案

短期 offload 先把完整工具结果写入 `refs/*.md`，再把每次 tool call/result 变成 `OffloadEntry`：

- `tool_call`：工具调用摘要
- `summary`：L1 生成的工具结果摘要
- `result_ref`：完整结果文件路径
- `tool_call_id`：原 provider tool call id
- `node_id`：L2 归属的 MMD 节点
- `score`：该摘要替代原文的可信/可替换程度

L2 prompt 再把这些 entries 合并进 MMD。关键约束是：

1. **节点可合并**：连续、意图相同的工具调用可以归为一个宏观节点。
2. **每个 tool_call_id 应有归属**：prompt 要求 `node_mapping` 覆盖新 `tool_call_id`，但代码不硬校验全覆盖；缺失时会用 mapping 中最频繁节点或 MMD 文本里的最新节点做 fallback，仍无节点时跳过。
3. **节点表达状态**：`done | doing | paused | blocked` 让模型知道当前焦点和雷区。
4. **图要受预算约束**：L2 prompt 要求 MMD 目标控制在 4000 字以内，summary 尽量小于 150 字；history MMD 注入还有 token ratio 降级。

MMD 会被注入消息流：

```text
<current_task_context>
当前活跃任务的 mermaid 流程图
任务目标
任务文件
节点状态说明
</current_task_context>
```

注入位置被刻意选择在最新 user message 后或尾部工具循环附近，并且不会插入到 assistant tool_use 与 tool_result 中间。

## 关键代码证据

- `src/offload/types.ts:12-30` — `OffloadEntry` schema，含 `node_id` / `result_ref` / `score`
- `src/offload/index.ts:434-445` — 完整工具结果写入 `refs/*.md`
- `src/offload/local-llm/prompts/l2-prompt.ts:9-30` — “用尽量少的字符表达尽量多的信息”、形状即语义、节点合并
- `src/offload/local-llm/prompts/l2-prompt.ts:36-54` — L2 输出 JSON schema 要求包含 `node_mapping`
- `src/offload/local-llm/parsers/l2-parser.ts:76-91` — 解析器读取已有 mapping，但不验证全覆盖
- `src/offload/pipelines/l2-mermaid.ts:220-266` — 缺失 mapping 时 fallback/skip 回填 node id
- `src/offload/hooks/after-tool-call.ts:214-222` — 将 active MMD 注入 `<current_task_context>`
- `src/offload/mmd-injector.ts:184-199` — 注入点选择策略
- `src/offload/mmd-injector.ts:228-260` — 避免拆散 tool_use/tool_result pair

## 为什么值得借鉴

### 1. 比摘要更可导航

普通摘要告诉模型“发生过什么”。MMD 还能告诉模型“现在在哪里、哪些节点已完成、哪个方向是 blocked、哪些信息属于同一阶段”。这对长任务 agent 比纯文本摘要更有操作性。

### 2. 让上下文保留更多“结构”，不是更多“原文”

它没有把更多原文塞进 context，而是把原文转成结构索引。LLM 看到的是可导航地图；完整证据还在 ref 文件里。这种结构密度比直接压缩成自然语言摘要更高。

### 3. 支持 LLM 自主展开

模型可以先基于 MMD 推理，只有缺细节时才通过 `read_file` 或 memory/conversation search 展开。单轮可能增加 1-3 次 recall 类 tool call，但可以减少重复探索和无意义大输出。

### 4. 更适合本地代理型 agent

本地文件作为 evidence store，MMD 作为任务地图，`read_file` 作为展开工具，这三者天然适合开发者本地代理。分布式 Web agent 也可以迁移，但要把 refs 换成对象存储/DB。

## 代价与局限

1. **L2 生成成本**：MMD 需要额外 LLM 调用，任务越碎，后台维护成本越高。
2. **工具调用次数转移**：从“重复读源文件/重跑命令”转成“memory search/read_file recall”，总体是否省成本取决于任务形态。
3. **符号图可能误导**：如果 L2 把节点状态或归因写错，主 LLM 会沿着错误地图推理。
4. **图本身也会膨胀**：L2 prompt 有 4000 字目标，history MMD 注入有 token ratio full/meta/skip 降级；active MMD 注入点只计算和 trace 预算，不做硬截断，长期多任务仍需要冷热归档。
5. **user-role 注入边界不够硬**：当前实现把 MMD 作为 user-role 文本插入；更严格的 SDK 可把它放入 runtime context section，避免和用户真实输入混淆。

## 对 agent-os 的转译

```text
ToolResultEvidenceStore
  保存原始工具输出，返回 evidence_ref

SymbolicContextMapBuilder
  输入 tool events + evidence refs + current task state
  输出 nodes / edges / node_to_evidence refs

ContextRenderer
  注入 compact map，不注入原文

recall_context
  支持 recall(node_id) / recall(evidence_ref)
```

关键不是照搬 Mermaid，而是保留“三段式”：**证据真值源 → 符号地图 → 按需展开工具**。

## 来源

- 完整分析：[[memory-system--tencentdb-agent-memory]]
- 源码版本：`v0.3.6-8-gf92b102`
