---
title: "Memory System — Claude Code"
category: L2
parent: "[[memory-system]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的记忆系统在上下文压缩之外引入了第三种知识载体：Session Memory File。它把长期稳定的任务状态、已知结论、关键文件路径等信息从对话消息流中剥离出来，沉淀为一个可落盘、可审阅的 Markdown 工作笔记。记忆的提取由后台 subagent 异步完成，不阻塞主会话；触发策略也做了细粒度控制，避免对短会话或不稳定会话过早激活。

## 架构分析

### 三层知识载体模型

Claude Code 明确区分了三种知识存储形式，而不是把所有信息都堆在消息历史里：

| 层次 | 载体 | 特点 |
|------|------|------|
| L1 | 当前消息历史 | 完整、精确，但有 token 限制 |
| L2 | 压缩后的对话快照 | 空间效率高，但信息有损 |
| L3 | Session Memory File | 长期稳定，可持久化，人工可读 |

Session Memory 专门存储那些"既不适合放在压缩快照里、又不能丢失"的信息：当前任务状态、已验证结论、关键文件与函数路径、错误和修正历史。

### 触发策略：shouldExtractMemory()

记忆提取不会在会话开始时立即激活。`shouldExtractMemory()` 是一个多维度的条件判断，综合考量：

- **总 token 规模**：会话体量足够大才有提取价值
- **上次提取后的增量**：防止频繁小量写入造成记忆文件碎片化
- **tool call 次数**：工具调用密集意味着任务实质性推进，值得记录
- **自然停顿点检测**：只在当前轮次处于自然停顿时提取，避免打断中间状态

这一多维触发策略的本质是：**记忆是昂贵的提炼操作，需要在"值得提炼"的时机才触发**。

### 文件化记忆：setupSessionMemoryFile()

`setupSessionMemoryFile()` 的执行流程：

1. 在项目目录下创建 memory 子目录
2. 创建以会话标识命名的 Markdown 文件
3. 写入结构化模板（包含各记忆类别的占位区域）
4. 用 `FileReadTool` 重新读取内容，注入当前上下文

关键设计：Session Memory 不是隐藏在内部状态里的黑盒，而是**一个可以被人类打开、审阅、手动编辑的 Markdown 文件**。这使得 agent 的"长期记忆"具有完全的可解释性和可干预性。

### 异步提取：后台 subagent 模式

记忆提取使用了 forked subagent 模式，关键依赖包括：

- `createSubagentContext`：为后台提取任务创建独立上下文
- `runForkedAgent`：在 fork 出的子 agent 中运行提取 prompt

这一设计使得：

- **主会话无感知**：用户与 agent 的交互不受记忆提取影响
- **提取独立失败**：后台提取失败不会影响主会话状态
- **提取可并发**：多轮提取可以在后台排队，不相互阻塞

`src/services/SessionMemory/prompts.ts` 定义了提取 prompt，决定了哪些信息会被归纳进 memory file。

### 与上下文压缩的协同

Session Memory 和 autoCompact 是互补而非替代关系：

- **压缩**处理的是"短期密集历史"的空间问题，输出是对话协议兼容的快照
- **Session Memory**处理的是"长期稳定结论"的提炼问题，输出是人类可读的工作笔记

两者在 `autoCompact.ts` 中有明确的协调逻辑：当处于 `session_memory` 模式时，自动压缩会被抑制，防止两套机制产生状态冲突。

### 关键代码路径

- `src/services/SessionMemory/sessionMemory.ts` — 记忆系统核心（触发策略、文件初始化、subagent 调度）
- `src/services/SessionMemory/prompts.ts` — 记忆提取 prompt（决定信息归纳的粒度与类别）
- `src/query.ts` — agent loop 主入口，session memory 在适当时机被调用

## 设计亮点

- **文件即记忆**：Session Memory 落盘为 Markdown 文件，而非隐藏在内存状态里；这使得记忆对用户完全透明，可以手动编辑、版本控制、跨会话共享
- **后台异步提取**：forked subagent 设计把记忆提取从主交互流中解耦，消除了"记忆更新卡交互"的体验问题
- **多维触发保护**：`shouldExtractMemory()` 的多条件组合有效防止了对短会话、不稳定会话的过早提取，避免生成低价值记忆碎片
- **三层知识载体明确分工**：消息历史 / 压缩快照 / Session Memory 各司其职，比"什么都塞进消息"的单层设计在长任务场景下鲁棒得多

### Auto-Dream 记忆归档

Claude Code 拥有所有已分析项目中唯一的**定期记忆归档机制**，位于 `src/services/autoDream/`。

**触发条件**（非固定时间，条件门控）：
- 距离上次归档 ≥ 24 小时
- 累计 ≥ 5 个新会话 transcript
- 在每轮结束的 stop hook 中检查

**归档四阶段**：
1. **Orient** — 扫描记忆目录结构，建立索引
2. **Gather** — 从近期会话 transcript 收集新信号
3. **Consolidate** — 写入/更新记忆文件
4. **Prune** — 删除过时条目

**执行机制**：
- Forked subagent 执行，不阻塞主会话
- `.consolidate-lock` 文件防止并发（PID + 1 小时超时）
- 只使用只读 bash 工具（grep/find/cat），不会意外修改代码
- UI 底部显示做梦进度

**配置**：GrowthBook feature flag `tengu_onyx_plover`，可通过 `settings.json` 的 `autoDreamEnabled` 覆盖。

**记忆三层架构**：
| 层级 | 机制 | 时机 |
|------|------|------|
| 即时提取 | 后台 forked subagent | 每轮结束 |
| 上下文压缩 | Message compaction | 接近 token 上限时 |
| 定期归档 | Auto-Dream | 24h + 5 会话后 |

## 局限性

- **记忆内容依赖 LLM 归纳质量**：提取 prompt 的输出质量受模型能力影响，可能遗漏重要信息或引入错误总结
- **文件粒度固定**：一个会话对应一个 memory file，对于跨多个子任务的长会话，所有记忆混在一个文件中，缺乏结构化索引
- **后台提取无反馈**：subagent 提取完成后，用户没有可见的通知；如果提取失败，用户无法感知，也无法手动触发补偿
- **跨会话归档有延迟**：Auto-Dream 的触发条件（24h + 5 会话）意味着短期内创建的记忆仍然分散在各会话 memory file 中；归档后碎片化问题大幅缓解，但实时合并仍不支持

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
