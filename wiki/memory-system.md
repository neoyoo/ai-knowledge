---
title: Memory System
aliases: [记忆系统, persistent memory, cross-session memory]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[context-management]]"
    type: feeds
  - target: "[[runtime-state]]"
    type: uses
sources: [claude-code, openharness, deer-flow]
---

## 一句话定义

跨会话的持久记忆 — 和 context 不同，活过对话结束，下次还在。

## 核心问题

- 什么值得记住，什么不值得？
- 记忆怎么存储（文件/数据库/向量）？
- 怎么在新对话开始时检索相关记忆？
- 记忆过时了怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow |
|------|------------|-------------|----------|
| 核心设计 | 在上下文压缩之外引入 Session Memory File（L3 知识载体），将长期稳定信息剥离为可落盘、可审阅的 Markdown 工作笔记，由后台 subagent 异步提取，不阻塞主会话 | 忠实移植 Claude Code 记忆模式：以 `~/.openharness/data/memory/{project-name}-{sha1}/` 为存储根，`MEMORY.md` 作为索引入口，prompt 构建时注入全文并通过纯词法匹配（约 20 行）选取最相关的最多 5 个专题文件注入 | 结构化 JSON 存储（`memory.json`）分三段：user（工作上下文）、history（近远期背景）、facts（带置信度的事实数组）；`MemoryMiddleware` 在 `after_agent` 钩子异步去抖更新；内置纠错/强化信号——检测"这不对"/"对"等词语自动调整事实置信度 |
| 关键特点 | 多维触发保护（`shouldExtractMemory()` 综合 token 规模/增量/tool call 次数/自然停顿点）；文件即记忆（人工可读、可编辑、可版本控制）；三层知识载体明确分工（消息历史/压缩快照/Session Memory） | 词法检索完全无外部依赖，无需 embedding 模型或向量数据库；文件系统存储人类可读、可直接编辑、可用 git 版本控制；首行元数据约定（160 字符描述）将检索开销降到极低 | 纠错/强化信号隐式反馈闭环（同类项目独有）；结构化 schema + 置信度排序注入，信噪比高于扁平 markdown；去抖异步更新不阻塞主循环；tiktoken 精确 token 预算控制 |
| 局限 | 记忆内容质量依赖 LLM 归纳；跨会话 memory file 不自动合并；后台提取无用户可见反馈 | 检索仅匹配标题和首行描述，正文内容对检索不可见；无后台自动提取机制，记忆写入完全依赖显式调用；无跨项目记忆共享能力 | 无向量检索（仅置信度排序，无语义相关性召回）；置信度排序非语境感知（高置信度事实不一定与当前对话相关）；无跨 agent 记忆共享 |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 文件存储（Markdown/JSON） | 记忆落盘为人类可读文件，词法检索，和代码放在一起 | 开发者工具、单项目 agent、需要 git 版本控制 | Claude Code、OpenHarness |
| 向量数据库 | 用 embedding 做语义检索，存 ChromaDB/Pinecone 等 | 大规模知识库、需要模糊匹配的客服/个人助手 | LangChain Memory、MemGPT |
| 结构化数据库（SQLite/Postgres） | schema 化存储，精确查询，强事务保证 | 用户画像、任务历史、有明确结构的偏好数据 | 自定义任务管理 agent |
| 混合（文件 + 向量） | Markdown 保留可读性，向量索引加速语义搜索 | 生产级长期助手，兼顾可调试性和检索质量 | 需自建 |

### 场景决策指南

**如果你在做开发者工具 / CLI agent → 文件存储**
记忆和代码一起在项目目录里，天然用 git 管理，用户能看到 agent 到底记了什么，出问题直接编辑文件。OpenHarness 这条路完全无外部依赖，词法匹配够用。

**如果你在做客服机器人，历史记录数万条 → 向量数据库**
词法匹配在这个规模下会漏掉大量语义相关内容。embedding 模型质量至关重要——先评估检索准确率再上线，别只看召回率。

**如果你在做任务管理 / 用户偏好系统 → 结构化数据库**
记忆本身有明确 schema（用户 ID、任务状态、时间戳），SQL 的精确查询和 ACID 保证比语义搜索更有价值。

**如果你在做长期个人助手，需要长达数月的记忆 → 混合方案**
Markdown 保留人类可读性和可干预性，向量索引解决规模化检索问题。代价是需要维护两套状态的一致性。

### 常见陷阱

- **什么都往记忆里塞**：检索噪音随记忆量线性增长，不加筛选的记忆比不记忆更危险。记忆写入前必须有过滤策略（Claude Code 的 `shouldExtractMemory()` 是个好例子）。
- **没有过期机制**：三个月前记录的用户偏好可能已经过时，过时记忆会系统性地误导 agent。至少要标注时间戳，定期审查高龄记忆。
- **把向量搜索当银弹**：embedding 质量差时，语义相近但实际无关的内容会被错误召回。上线前必须用真实查询做检索质量评估，不能假设 embedding 模型自动解决一切。
- **记忆写入没有反馈**：Claude Code 的后台提取在失败时用户无感知。生产系统需要监控记忆提取成功率，否则故障会悄悄积累。

## L2 详情

- [[memory-system--claude-code]]
- [[memory-system--openharness]]
- [[memory-system--deer-flow]]
