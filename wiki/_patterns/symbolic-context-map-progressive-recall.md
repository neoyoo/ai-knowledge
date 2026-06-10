---
title: Symbolic Context Map + Progressive Recall
aliases: [符号上下文图, symbolic context map, progressive recall, evidence-backed context map]
kind: pattern
created: 2026-06-09
updated: 2026-06-09
concepts_involved: [[context-management]], [[memory-system]], [[tool-system]], [[prompt-system]]
reference_implementations: [tencentdb-agent-memory]
status: emerging
relations:
  - target: "[[tencentdb-agent-memory--mmd-symbolic-context-map]]"
    type: extracted_from
  - target: "[[tool-metadata-driven-context-lifecycle]]"
    type: complements
---

## 一句话定义

把大段历史和工具结果压缩成 LLM 可读的**符号地图**，保留节点、状态、边和证据引用；默认只注入地图，细节由 LLM 通过 recall/read 工具按需展开。

## 触发问题

长任务 agent 的上下文膨胀通常不是因为用户消息太多，而是因为工具结果、文件读取、搜索结果、子任务报告不断进入 message history。直接塞原文会撑爆 context；纯摘要又会丢掉证据和任务拓扑。

需要一种中间层：

- 比自然语言摘要更结构化；
- 比原文更省 token；
- 保留可召回的 evidence handle；
- 让 LLM 自己判断什么时候需要展开细节。

## 参与的概念

- [[context-management]] — 控制哪些内容进入当前 context，哪些只作为 handle
- [[memory-system]] — 保存跨轮/跨会话的证据、场景、persona、原始对话
- [[tool-system]] — 提供 `recall_context` / `read_file` / memory search 等展开工具
- [[prompt-system]] — 将符号地图渲染为 LLM 可理解、边界清楚的 context section

## 核心设计

### 1. Evidence Store 保存原文真值源

大工具结果、长文件、搜索结果、子 agent 报告不直接长期驻留 context，而是保存为可寻址证据：

```text
evidence_ref:
  id: ref_2026_06_09_001
  source: tool_result
  tool_name: read_file
  size_tokens: 18342
  preview: "src/agentos/runtime/query_loop.py ..."
  uri: refs/2026-06-09-read_file-001.md
```

本地 profile 可以是 filesystem/SQLite；Web 分布式 profile 可以是 object store + DB index。

### 2. Symbolic Context Map 维护结构索引

把多个 evidence refs 和运行事件聚合为节点：

```text
Node:
  id: N4
  label: "compression budget hardening"
  status: doing
  summary: "已确认 tool_result cap 只做 nudge，缺 evidence recall"
  evidence_refs: [ref_001, ref_004, ref_009]
  timestamp: 2026-06-09T10:31:00Z
```

图不一定要用 Mermaid。Mermaid/MMD 的价值在于它是 LLM 熟悉的紧凑文本格式；SDK 也可以用 Markdown outline、XML、JSON-like DSL。关键是有节点、状态、边、证据引用和预算控制。

### 3. Context Renderer 默认只注入地图

LLM 可见 context 中放：

- 当前任务地图
- 每个节点的极短 summary
- doing/blocked/done 状态
- 可展开 handle
- 明确规则：地图是 runtime context，不是用户新指令

不要默认注入所有 evidence 原文。地图用于方向感，原文用于必要时校验。

### 4. Recall 工具按需展开

提供少量受预算控制的展开工具：

- `recall_context(node_id=...)`：展开某个节点关联的证据摘要或原文片段
- `recall_context(evidence_ref=...)`：展开单个证据
- `search_memory(query=...)`：跨记忆检索
- `read_file(path=...)`：本地 evidence 或 scene block 读取

每轮必须有调用预算，例如 memory/evidence recall 合计最多 3 次。这个预算最好由工具层或 hook 层硬执行；只写进 prompt/description 能引导模型，但不能防止搜索循环。

## 参考实现：TencentDB Agent Memory

TencentDB Agent Memory 的短期 offload 链路是该模式的直接参考：

- 完整工具结果写入 `refs/*.md`
- L1 摘要进入 `offload-<sessionId>.jsonl`
- L2 生成 Mermaid/MMD 认知状态机
- prompt 要求 `node_mapping` 将新 `tool_call_id` 映射到节点；代码读取 mapping 并提供 fallback/skip，覆盖率需要观测
- active MMD 注入 `<current_task_context>`
- memory tools guide 提醒模型必要时调用 `tdai_memory_search` / `tdai_conversation_search` / `read_file`

详见 [[memory-system--tencentdb-agent-memory]] 与 [[tencentdb-agent-memory--mmd-symbolic-context-map]]。

## 和 LLM-Driven Context Lifecycle 的关系

[[tool-metadata-driven-context-lifecycle]] 关注“某个 tool_result 何时 free / recall”。本模式关注“很多 tool_result 和历史事件如何形成可导航地图”。

两者可以组合：

```text
tool_result too large
  → save evidence
  → replace message with ref/nudge
  → update symbolic context map
  → LLM later recall(node/evidence) if needed
```

前者是生命周期协议，后者是高密度导航层。

## 设计权衡

| 方案 | 优点 | 代价 |
|---|---|---|
| 只做摘要 | 实现简单，token 低 | 丢拓扑，缺证据 handle，难按需展开 |
| 只做 evidence refs | 不丢原文，易审计 | LLM 缺方向感，不知道该展开哪个 ref |
| 符号地图 + evidence refs | 结构密度高，支持自主展开 | 需要 map builder、预算控制和质量观测 |

## 适用场景

- 本地代码代理，工具结果和文件读取密集
- 长任务 agent，需要跨几十轮保持方向感
- 多 agent 汇总，主 agent 只需要结构化进度与证据 handle
- Web 分布式 agent，需要把大 payload 放到 object store，而不是 SSE/HTTP message

## 不适用场景

- 单轮问答或短对话
- 工具结果天然很小
- 模型能力较弱，无法稳定使用 handle/recall
- 对延迟极敏感，无法接受额外后台 map 更新或 recall tool call

## 常见陷阱

- **地图当真相**：符号地图是索引，不是真值源。回答关键事实前应能回到 evidence 校验。
- **没有调用预算**：progressive recall 会把 token 压力转成 tool-call 压力，必须限制每轮搜索次数。
- **覆盖率不可见**：如果 map builder 没有统计 node 覆盖率、fallback 率和 skip 率，地图会看起来完整，实际却可能有证据没有归属。
- **handle 太多**：地图里满屏 ref 会污染注意力。节点应聚合 evidence，而不是每条证据单独暴露。
- **上下文边界不清**：地图必须标为 runtime context，不能让模型误以为是用户新输入。
- **分布式场景沿用本地路径**：Web profile 中 evidence URI 必须做权限校验和租户隔离，不能把本地绝对路径暴露给模型或用户。

## 迁移 checklist

1. 定义 `EvidenceStore`：保存原文、返回 ref、支持 TTL/权限。
2. 定义 `SymbolicContextMap` schema：node/status/summary/edges/evidence_refs。
3. 在工具结果写入 message history 前接入 evidence offload。
4. 在 context renderer 中加入 map section，明确“非用户输入”。
5. 扩展 recall 工具，支持 node/evidence 两种粒度。
6. 设 per-turn recall budget，并记录 recall metrics。
7. 定期评估 map 质量：node 覆盖率、fallback/skip 率、错误归因、recall 命中率、额外 tool call 次数。

## 相关概念

- [[context-management]]
- [[memory-system]]
- [[tool-system]]
- [[prompt-system]]
- [[tool-metadata-driven-context-lifecycle]]
