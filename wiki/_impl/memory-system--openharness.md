---
title: "Memory System — OpenHarness"
category: L2
parent: "[[memory-system]]"
source: openharness
source_version: "v0.1.9-51-g9b2efd7"
confidence: high
created: 2026-04-06
updated: 2026-06-10
---

## 概述

OpenHarness 的记忆系统已经从早期 Claude Code 风格的 `MEMORY.md + 专题文件`，演进为**文件可读性 + 结构化 frontmatter + 轻量检索评分 + 后台整理**的本地记忆层。它仍以 `~/.openharness/data/memory/{project-name}-{sha1hash}/` 为存储根目录、`MEMORY.md` 为索引入口，但专题文件现在支持 schema-v1 frontmatter、type/scope/category/importance/source/signature/ttl/disabled/supersedes/tags 等字段；检索会同时看元数据和 body preview，并叠加 importance、usage count、recency；后台有 auto-extract 和 auto-dream consolidation。

## 架构分析

### 存储结构

记忆文件存储在 `~/.openharness/data/memory/{project-name}-{sha1hash}/`，路径由 `memory/paths.py` 管理。项目名加 SHA1 哈希确保不同项目之间完全隔离。目录下 `MEMORY.md` 是必须存在的索引文件，其余 `.md` 文件为专题记忆文件，每个文件对应一个主题域。`memory/memdir.py` 负责目录的初始化和文件管理。

### Prompt 注入流程

在每次构建 prompt 时执行两步注入：(1) 将 `MEMORY.md` 注入为全局上下文索引（受 `max_entrypoint_lines` / `max_entrypoint_bytes` 限制）；(2) 调用 `memory/search.py` 对当前用户输入进行词法匹配，从所有专题文件中选出最多 `max_files`（默认 5）个相关文件，并通过 `format_relevant_memories()` 注入每个文件前 `max_chars` 字符（默认 8000）。注入发生在 prompt-build 时，不是运行时动态检索。

### 结构化 memory schema

`src/openharness/memory/schema.py` 定义 schema-v1 frontmatter：

- `schema_version`、`id`、`name`、`description`
- `type: user|feedback|project|reference`
- `scope: private|project|team`
- `category`、`importance`、`source`、`signature`
- `created_at`、`updated_at`、`ttl_days`
- `disabled`、`supersedes`、`tags`

同文件还提供 signature 计算、TTL/过期判断、`MEMORY.md` 行数和字节上限、durable memory policy。`signature` 用内容/type/category 的规范化 hash 做重复检测；`ttl_days` 和 `disabled` 让记忆可以“停用而非删除”，适合自动整理。

### 词法检索实现（已升级）

`memory/search.py` 仍保持零 embedding 依赖，但不再只看标题和首行。它会：

- 从 query 中抽取 ASCII token 和 Han 字符，支持中文检索；
- 用 frontmatter 的 title/description 做 metadata match；
- 用 `scan_memory_files()` 生成的 `body_preview` 做正文预览匹配；
- metadata 命中权重 2x，body 命中 1x；
- 叠加 `importance * 0.4`、usage count、14/30 天 recency boost。

`memory/scan.py` 会解析 YAML frontmatter，跳过 `disabled` 和过期记忆，正文预览最多 300 字符。`memory/usage.py` 用 `usage_index.json` 记录 recalled memory 的 `use_count` / `last_used_at`，被检索出的记忆会在 prompt 构建时更新 usage。

### CRUD 操作

`memory/manager.py` 提供 `add_memory_entry()` 和 `remove_memory_entry()` 用于 `MEMORY.md` 索引的增删。专题文件的创建和编辑直接通过文件系统操作，无数据库或专有格式。`memory/scan.py` 负责扫描记忆目录，枚举所有可用的专题文件供检索使用。

### Auto-extract 与 Auto-Dream

`QueryEngine.submit_message()` 在每轮结束后依次执行 session memory update、durable memory extraction、auto-dream schedule：

- `src/openharness/engine/query_engine.py:170-184` — turn 结束后写 session checkpoint memory；
- `query_engine.py:186-210` — 可选 auto-extract，把当前 turn 中值得长期保存的事实写入 memory；
- `query_engine.py:275-278` — finally 中更新 session memory、提取 durable memory、调度 auto-dream。

`src/openharness/services/autodream/service.py` 在满足时间/会话数门槛后启动后台 dream task：

- consolidation lock 防止并发整理；
- preview 模式只产出 patch plan；
- 非 preview 模式先备份 memory 目录，再通过 prompt 约束子进程 agent 只修改 memory 目录，并用 lock、backup、diff metadata、completion listener 追踪实际改动；
- task failed/killed 或 preview 时 rollback consolidation lock；
- completion listener 记录 added/changed/removed/touched 文件。

`autodream/prompt.py` 把整理任务拆成 Orient → Gather recent signal → Consolidate → Prune/index 四阶段，并强制 evidence discipline、分类、隐私和 stale/snapshot 标注。

### 关键代码路径

- `memory/paths.py` — 存储路径计算，`{project-name}-{sha1hash}` 目录命名逻辑
- `memory/manager.py` — `add_memory_entry()`、`remove_memory_entry()` CRUD 接口
- `memory/scan.py` — 记忆目录扫描，枚举可检索文件列表
- `memory/search.py` — metadata/body/importance/usage/recency 加权的零 embedding 检索
- `memory/memdir.py` — 记忆目录初始化与文件管理
- `memory/schema.py` — schema-v1 frontmatter、signature、TTL、entrypoint 上限、durable memory policy
- `memory/usage.py` — recalled memory usage index 与 stale candidate 查找
- `services/autodream/service.py` — auto-dream 调度、lock、backup、rollback、diff metadata
- `services/autodream/prompt.py` — memory consolidation prompt
- `engine/query_engine.py` — turn 结束后的 session memory、auto-extract、auto-dream 调用链

## 设计亮点

- 保留 Claude Code 的 `MEMORY.md` + 专题文件心智模型，同时用 schema/frontmatter 补足类型、范围、TTL、去重和停用语义
- 词法检索完全无外部依赖，无需 embedding 模型或向量数据库，离线可用；新版已支持 metadata/body/importance/usage/recency 多信号评分
- 文件系统作为存储后端，记忆内容人类可读、可直接编辑、可用 git 版本控制
- Auto-Dream 的 backup/rollback/preview/diff 机制让“后台整理记忆”可审计、可回滚，避免无保护地让子 agent 改长期记忆
- SDK runtime 与 `ohmo` 产品壳分离：OpenHarness 提供通用 memory/core，ohmo 使用独立 workspace/personal memory，不把产品人格写进 SDK

## 局限性

- 无向量/嵌入搜索，语义相近但词汇不同的查询仍可能漏召回（如"token 压缩"无法稳定匹配含"context compaction"的文件）
- body preview 只取正文前 300 字符，长文后半段仍对检索不可见
- Auto-extract / Auto-Dream 依赖 LLM 归纳质量，仍需人工 review 和质量监控
- Auto-Dream 子进程通过 `openharness --dangerously-skip-permissions --print` 或 `ohmo --workspace ...` 在项目 cwd 下运行；“只改 memory 目录”主要是 prompt/tool constraint 加 backup/diff/lock 护栏，不是 OS 级硬沙箱，也不是数据库事务
- 记忆按项目隔离，无跨项目记忆共享能力，通用领域知识无法在项目间复用

## 来源

- 源码版本：`v0.1.9-51-g9b2efd7`
- 分析深度：源码级
