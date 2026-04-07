---
title: "Memory System — DeerFlow"
category: L2
parent: "[[memory-system]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 拥有目前所有对比项目中最完整的记忆系统。以结构化 JSON 文件（`memory.json`）为存储后端，分三段组织：`user`（工作上下文/个人背景/当前关注）、`history`（近期/较早/长期背景）、`facts`（带置信度的事实数组）。`MemoryMiddleware` 在 `after_agent` 钩子上挂载，通过 `MemoryUpdateQueue` 异步去抖更新，避免阻塞主循环。注入时按置信度排序，在 token 预算内（默认 2000，tiktoken 精确计数）注入系统提示。最独特的是内置纠错/强化信号：检测用户消息中的"that's wrong"/"不对"或"perfect"/"完全正确"，通知 LLM 调整相关事实的置信度。

## 架构分析

### 结构化存储 Schema

`memory.json` 固定三段结构：

```
{
  "user": {
    "workContext": "...",
    "personalContext": "...",
    "topOfMind": "..."
  },
  "history": {
    "recentMonths": "...",
    "earlierContext": "...",
    "longTermBackground": "..."
  },
  "facts": [
    { "id": "...", "content": "...", "category": "...", "confidence": 0.9 }
  ]
}
```

相比 Claude Code 的扁平 markdown 文件，结构化 schema 使得按类别查询、按置信度排序成为可能，也便于未来接入向量索引。

### 异步去抖更新（MemoryUpdateQueue）

`MemoryMiddleware` 在 `after_agent` 钩子触发后，将当前对话推入 `MemoryUpdateQueue` 而非立即更新。队列有去抖窗口——短时间内多次触发只执行一次更新，避免高频对话时重复调用 LLM。`MemoryUpdater` 消费队列，用 `MEMORY_UPDATE_PROMPT` 指引 LLM 从对话中提取新事实或修订已有事实。

### 纠错/强化信号

在调用 `MemoryUpdater` 前，系统扫描用户最近消息：
- 检测到"that's wrong"/"不对"等否定词 → 传入 correction hint，提示 LLM 降低相关事实置信度或删除；
- 检测到"perfect"/"完全正确"等肯定词 → 传入 reinforcement hint，提示 LLM 提升相关事实置信度。

这使记忆系统具备隐式反馈闭环，无需用户显式管理。

### 注入与 Token 预算

`format_memory_for_injection()` 在构建系统提示时：
1. 从 `facts` 数组按 confidence 降序排列；
2. 用 tiktoken 累计 token 数，超过预算（默认 2000）则截断；
3. 将 user/history 摘要与高置信度 facts 一起包装在 `<memory>` XML 标签内注入。

### Per-Agent 命名空间

自定义 agent 有各自独立的 `memory.json`，通过 agent 名称作为路径前缀隔离。文件级缓存用 mtime 判断失效，避免重复读取未变更的文件。

### 关键代码路径

- `deerflow/agents/memory/storage.py` — `FileMemoryStorage`，JSON 读写与 mtime 缓存
- `deerflow/agents/memory/updater.py` — `MemoryUpdater`，LLM 提取事实 + 纠错/强化逻辑
- `deerflow/agents/memory/queue.py` — `MemoryUpdateQueue`，去抖异步队列
- `deerflow/agents/middlewares/memory_middleware.py` — after_agent 钩子挂载，信号检测

## 设计亮点

- **纠错/强化信号**：隐式反馈闭环是所有对比项目中独有的设计，用户无需主动管理记忆，系统自动感知对话中的确认/否定信号并调整置信度；
- **结构化 schema + 置信度**：facts 数组按置信度排序注入，高质量信息优先进入上下文窗口，比扁平 markdown 信噪比更高；
- **去抖异步更新**：记忆更新不阻塞主循环，且高频场景下合并触发，LLM 调用次数可控；
- **Token 预算精确控制**：tiktoken 计数 + 硬截断，确保记忆注入不超出系统提示预算，不影响主任务 token 分配。

## 局限性

- **无向量检索**：TF-IDF 检索已在规划但尚未落地，当前仅靠置信度排序，无法按上下文语义相关性召回最相关记忆；
- **置信度排序非语境感知**：高置信度事实不一定与当前对话相关，注入可能引入无关噪声；
- **无跨 agent 记忆共享**：每个 agent 的记忆完全隔离，多 agent 协作场景下无法共享用户知识，需手动同步。

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
