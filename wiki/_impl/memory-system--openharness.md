---
title: "Memory System — OpenHarness"
category: L2
parent: "[[memory-system]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness 的记忆系统是对 Claude Code 记忆模式的忠实移植：以 `~/.openharness/data/memory/{project-name}-{sha1hash}/` 为存储根目录，`MEMORY.md` 作为索引入口，按项目隔离。每次构建 prompt 时，系统注入完整的 `MEMORY.md` 内容，并通过纯词法匹配将当前用户输入与各专题文件的标题和首行做词频对比，选取最相关的最多 5 个文件一并注入。整套检索逻辑约 20 行，刻意回避向量化和嵌入式搜索。

## 架构分析

### 存储结构

记忆文件存储在 `~/.openharness/data/memory/{project-name}-{sha1hash}/`，路径由 `memory/paths.py` 管理。项目名加 SHA1 哈希确保不同项目之间完全隔离。目录下 `MEMORY.md` 是必须存在的索引文件，其余 `.md` 文件为专题记忆文件，每个文件对应一个主题域。`memory/memdir.py` 负责目录的初始化和文件管理。

### Prompt 注入流程

在每次构建 prompt 时执行两步注入：(1) 将 `MEMORY.md` 全文注入（受 `max_entrypoint_lines` 限制），作为全局上下文索引；(2) 调用 `memory/search.py` 对当前用户输入进行词法匹配，从所有专题文件中选出最多 `max_files`（默认 5）个相关文件并全文注入。注入发生在 prompt-build 时，不是运行时动态检索。

### 词法检索实现

`memory/search.py` 的检索逻辑约 20 行：用正则从查询字符串提取 token 集合，对每个专题文件读取标题（文件名）和首行描述（最多 160 字符），统计 token 出现次数作为相关性分数，按分数降序选取 top-N 文件。检索不读取文件正文，仅依赖元数据（标题 + 首行），这是有意的速度换精度权衡。

### CRUD 操作

`memory/manager.py` 提供 `add_memory_entry()` 和 `remove_memory_entry()` 用于 `MEMORY.md` 索引的增删。专题文件的创建和编辑直接通过文件系统操作，无数据库或专有格式。`memory/scan.py` 负责扫描记忆目录，枚举所有可用的专题文件供检索使用。

### 关键代码路径

- `memory/paths.py` — 存储路径计算，`{project-name}-{sha1hash}` 目录命名逻辑
- `memory/manager.py` — `add_memory_entry()`、`remove_memory_entry()` CRUD 接口
- `memory/scan.py` — 记忆目录扫描，枚举可检索文件列表
- `memory/search.py` — 纯词法相关性计算，约 20 行核心逻辑
- `memory/memdir.py` — 记忆目录初始化与文件管理

## 设计亮点

- 忠实移植 Claude Code 的 `MEMORY.md` + 专题文件模式，用户心智模型与 Claude Code 对齐
- 词法检索完全无外部依赖，无需 embedding 模型或向量数据库，离线可用
- 文件系统作为存储后端，记忆内容人类可读、可直接编辑、可用 git 版本控制
- 首行元数据约定（160 字符描述）将检索开销降到极低，单次扫描无 I/O 放大

## 局限性

- 检索仅匹配标题和首行描述，正文内容对检索完全不可见——专题文件内容越丰富，检索精度越低
- 无向量/嵌入搜索，语义相近但词汇不同的查询无法匹配（如"token 压缩"无法匹配含"context compaction"的文件）
- 无后台自动提取机制，记忆的写入完全依赖用户或工具显式调用，不像 Claude Code 通过 forked subagent 自动提炼
- 记忆按项目隔离，无跨项目记忆共享能力，通用领域知识无法在项目间复用

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
