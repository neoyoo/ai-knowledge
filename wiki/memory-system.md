---
title: Memory System
aliases: [记忆系统, persistent memory, cross-session memory]
category: L1
created: 2026-04-06
updated: 2026-04-08
relations:
  - target: "[[context-management]]"
    type: feeds
  - target: "[[runtime-state]]"
    type: uses
sources: [claude-code, openharness, deer-flow, hermes-agent]
---
	
## 一句话定义

跨会话的持久记忆 — 和 context 不同，活过对话结束，下次还在。

## 核心问题

- 什么值得记住，什么不值得？
- 记忆怎么存储（文件/数据库/向量）？
- 怎么在新对话开始时检索相关记忆？
- 记忆过时了怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|-------------|
| 核心设计 | 在上下文压缩之外引入 Session Memory File（L3 知识载体），将长期稳定信息剥离为可落盘、可审阅的 Markdown 工作笔记，由后台 subagent 异步提取，不阻塞主会话 | 忠实移植 Claude Code 记忆模式：以 `~/.openharness/data/memory/{project-name}-{sha1}/` 为存储根，`MEMORY.md` 作为索引入口，prompt 构建时注入全文并通过纯词法匹配（约 20 行）选取最相关的最多 5 个专题文件注入 | 结构化 JSON 存储（`memory.json`）分三段：user（工作上下文）、history（近远期背景）、facts（带置信度的事实数组）；`MemoryMiddleware` 在 `after_agent` 钩子异步去抖更新；内置纠错/强化信号——检测"这不对"/"对"等词语自动调整事实置信度 | 双层架构：内置层（`MEMORY.md` + `USER.md`，始终存在）+ 外部 provider 层（8 个可插拔 provider，一次只激活一个）；session 启动时烘焙冻结快照注入系统提示，中途写入不更新缓存；Nudge 机制每隔 N 轮 fork 独立后台 review agent 自主写入记忆 |
| 关键特点 | 多维触发保护（`shouldExtractMemory()` 综合 token 规模/增量/tool call 次数/自然停顿点）；文件即记忆（人工可读、可编辑、可版本控制）；三层知识载体明确分工（消息历史/压缩快照/Session Memory） | 词法检索完全无外部依赖，无需 embedding 模型或向量数据库；文件系统存储人类可读、可直接编辑、可用 git 版本控制；首行元数据约定（160 字符描述）将检索开销降到极低 | 纠错/强化信号隐式反馈闭环（同类项目独有）；结构化 schema + 置信度排序注入，信噪比高于扁平 markdown；去抖异步更新不阻塞主循环；tiktoken 精确 token 预算控制 | Context Fencing（`<memory-context>` 标签 + "NOT new user input" 标注）防止召回内容被误识别为用户输入；注入防护扫描（11 类威胁模式：prompt injection + 凭证外泄 + 零宽字符）；FTS5 跨 session 搜索 + 并发 LLM 摘要两级召回；单 provider 约束防止工具 schema 膨胀 |
| 局限 | 记忆内容质量依赖 LLM 归纳；跨会话 memory file 不自动合并；后台提取无用户可见反馈 | 检索仅匹配标题和首行描述，正文内容对检索不可见；无后台自动提取机制，记忆写入完全依赖显式调用；无跨项目记忆共享能力 | 无向量检索（仅置信度排序，无语义相关性召回）；置信度排序非语境感知（高置信度事实不一定与当前对话相关）；无跨 agent 记忆共享 | 冻结快照导致本轮写入的记忆下次 session 才生效；单 provider 约束无法同时激活多个外部后端；内置层全量注入无语义排序，记忆条目增多信噪比下降；后台 review agent 成本和延迟在高频短对话场景不可控 |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 文件存储（Markdown/JSON） | 记忆落盘为人类可读文件，词法检索，和代码放在一起 | 开发者工具、单项目 agent、需要 git 版本控制 | Claude Code、OpenHarness |
| 向量数据库 | 用 embedding 做语义检索，存 ChromaDB/Pinecone 等 | 大规模知识库、需要模糊匹配的客服/个人助手 | LangChain Memory、MemGPT |
| 结构化数据库（SQLite/Postgres） | schema 化存储，精确查询，强事务保证 | 用户画像、任务历史、有明确结构的偏好数据 | 自定义任务管理 agent |
| 混合（文件 + 向量） | Markdown 保留可读性，向量索引加速语义搜索 | 生产级长期助手，兼顾可调试性和检索质量 | 需自建 |
| 双层（内置始终在线 + 外部可插拔） | 内置层保底可靠性，外部层按需扩展能力（向量/图谱/用户建模），二者独立维护 | 需要多种检索后端灵活切换、又不想每个部署都自建的通用助手 | Hermes Agent |

### 场景决策指南

**如果你在做开发者工具 / CLI agent → 文件存储**
记忆和代码一起在项目目录里，天然用 git 管理，用户能看到 agent 到底记了什么，出问题直接编辑文件。OpenHarness 这条路完全无外部依赖，词法匹配够用。

**如果你在做客服机器人，历史记录数万条 → 向量数据库**
词法匹配在这个规模下会漏掉大量语义相关内容。embedding 模型质量至关重要——先评估检索准确率再上线，别只看召回率。

**如果你在做任务管理 / 用户偏好系统 → 结构化数据库**
记忆本身有明确 schema（用户 ID、任务状态、时间戳），SQL 的精确查询和 ACID 保证比语义搜索更有价值。

**如果你在做长期个人助手，需要长达数月的记忆 → 混合方案**
Markdown 保留人类可读性和可干预性，向量索引解决规模化检索问题。代价是需要维护两套状态的一致性。

**如果你需要可换的记忆后端，又关注前缀缓存成本 → 双层 + 冻结快照（Hermes Agent 模式）**
内置层提供零依赖的基础记忆，外部 provider 通过统一 ABC 热插拔（Honcho/Hindsight/Mem0/Holographic 等 8 种）。冻结快照确保系统提示前缀每个 session 只变一次，Anthropic Prompt Cache 命中率最大化。注意：如果你的对话是长 session 且中途经常获得重要新信息，冻结快照的"下次才生效"设计会让 agent 在当前 session 内无法利用刚学到的内容。

### 常见陷阱

- **什么都往记忆里塞**：检索噪音随记忆量线性增长，不加筛选的记忆比不记忆更危险。记忆写入前必须有过滤策略（Claude Code 的 `shouldExtractMemory()` 是个好例子）。
- **没有过期机制**：三个月前记录的用户偏好可能已经过时，过时记忆会系统性地误导 agent。至少要标注时间戳，定期审查高龄记忆。
- **把向量搜索当银弹**：embedding 质量差时，语义相近但实际无关的内容会被错误召回。上线前必须用真实查询做检索质量评估，不能假设 embedding 模型自动解决一切。
- **记忆写入没有反馈**：Claude Code 的后台提取在失败时用户无感知。生产系统需要监控记忆提取成功率，否则故障会悄悄积累。
- **记忆内容是注入攻击的高价值目标**：记忆最终会注入系统提示，一旦被植入恶意指令后果极严重。Hermes Agent 的 `_scan_memory_content()` 在任何写入前扫描 11 类威胁模式（prompt injection 短语、curl/wget 带 secrets 外泄模式、零宽字符）。自建记忆系统必须有等效的写前校验，不能假设 agent 只会记"正常内容"。
- **后台 review agent 的成本失控**：Nudge 机制（Hermes Agent）fork 独立 AIAgent 后台审查记忆是优雅的设计，但在高频短对话场景（如群聊 gateway）nudge_interval 设小了会大量并发 review agent，token 消耗难以预估。上线前需根据预期对话频率仔细调参，并监控 review agent 的调用量。
- **多 provider 幻觉**：允许同时激活多个记忆后端看似功能丰富，实际上会导致工具 schema 膨胀、LLM 在记忆工具选择上混乱，还要处理多后端数据一致性。Hermes Agent 用单 provider 约束（`_has_external` flag）明确拒绝第二个外部 provider，是工程上的务实取舍。

### 记忆生命周期管理

除了"存哪里"和"怎么检索"，还有一个常被忽略的问题：**记忆怎么维护？**

| 策略 | 做法 | 代表 | 适合 |
|------|------|------|------|
| 无维护 | 只写不管 | OpenHarness | 短期项目、实验 |
| 手动维护 | 用户自己编辑/删除 | Claude Code MEMORY.md | 个人工具 |
| 信号调整 | 纠正/强化自动改置信度 | DeerFlow | 需要从对话中学习的系统 |
| 定期归档 | 后台 agent 定期整理+剪枝 | Claude Code Auto-Dream | 长期运行的生产 agent |
| 时态过期 | 事实自带有效期，过期自动失效 | MiroFish Zep | 信息时效性强的场景 |
| 轮次触发 review | 每隔 N 轮 fork 独立 AIAgent 在后台自主回顾并写入记忆，主对话不感知 | Hermes Agent Nudge | 需要持续被动积累记忆、不想依赖 agent 主动触发的场景 |

**关键教训**：没有维护的记忆系统最终会变成垃圾堆。记忆越多 ≠ 越有用——过时、矛盾、低质量的记忆会主动伤害 agent 表现。

**Claude Code 的 Auto-Dream 是目前最成熟的自动维护方案**：条件触发（24h + 5 会话），4 阶段归档（Orient → Gather → Consolidate → Prune），forked subagent 不阻塞主流程。

**Hermes Agent 的 Nudge 是被动积累的补充路径**：不依赖 agent 主动判断"应该写记忆"，而是每隔固定轮次后台 fork review agent 自主决策。适合对话节奏不规律、主 agent 记忆写入率偏低的场景。两种机制不互斥：Auto-Dream 负责定期整理，Nudge 负责持续捕获增量——如果自建生产 agent，可以组合使用。

### 外部 provider 的能力谱系（Hermes Agent 实践）

Hermes Agent 的 8 个外部 provider 覆盖了当前主流的记忆后端技术路线，提供了难得的横向对比视角：

| Provider | 核心技术 | 独特能力 |
|----------|---------|---------|
| Honcho | AI-native 用户建模 | Dialectic Q&A 迭代更新用户表示；三种 recall_mode；cost cadence 控制 API 频率 |
| Hindsight | 知识图谱 + 多策略检索 | 语义搜索 + 关键词 + 实体图遍历 + reranking；支持 local 嵌入（无需云 API） |
| Holographic | HRR 向量 + SQLite | 结构化事实（entity/category/tag/trust_score）+ HRR 向量（`hrr_dim=1024`）；`fact_feedback` 动态调整 trust |
| Mem0 | 服务端 LLM 抽取 | 自动去重；内置熔断器（连续失败 5 次冷却 120 秒） |
| Supermemory | container 语义存储 | 按 tag 划分容器；session_end 批量 ingest 全对话 |
| RetainDB | SQLite write-behind | crash-safe 异步写队列；dialectic 合成 + agent self-model（SOUL.md persona） |
| ByteRover | 分层上下文树 | `viking://` URI 两级检索（fuzzy text → LLM-driven）；local-first 可选云同步 |
| OpenViking | Volcengine 上下文 DB | 三级 context loading（L0/L1/L2）；`atexit` 保证进程崩溃时仍提交 pending sessions |

这张表的价值不在于选哪个，而在于揭示了不同技术路线的本质差异：**用户建模（Honcho）vs 事实图谱（Hindsight/Holographic）vs 全对话归档（Supermemory/OpenViking）vs 可靠性优先（Mem0 熔断器 / RetainDB write-behind）**。自建外部 provider 时，先想清楚你的核心诉求属于哪个象限。

## L2 详情

- [[memory-system--claude-code]]
- [[memory-system--openharness]]
- [[memory-system--deer-flow]]
- [[memory-system--hermes-agent]]
