---
title: Memory System
aliases: [记忆系统, persistent memory, cross-session memory]
category: L1
created: 2026-04-06
updated: 2026-04-15
relations:
  - target: "[[context-management]]"
    type: feeds
  - target: "[[runtime-state]]"
    type: uses
  - target: "[[tool-system]]"
    type: depends_on
    evidence: "AgentScope LTM 的 record_to_memory / retrieve_from_memory 以工具函数形式暴露，LLM 通过工具调用自主管理长期记忆；记忆能力与工具系统深度耦合，LTM 本质是一组特殊工具"
  - target: "[[prompt-system]]"
    type: feeds
    evidence: "记忆召回最终会注入 prompt / messages；memory-context fencing 与 prompt 注入边界决定召回内容是否会被误当成用户指令"
  - target: "[[multi-agent]]"
    type: supports
    evidence: "AgentScope Mark 系统允许同一 memory 实例服务多个 agent/角色，通过 mark 过滤隔离各自上下文，为 multi-agent 编排中的工具链与主对话流分离提供了轻量原语"
sources: [claude-code, openharness, deer-flow, hermes-agent, mempalace, agentscope]
---
	
## 一句话定义

跨会话的持久记忆 — 和 context 不同，活过对话结束，下次还在。

## 核心问题

- 什么值得记住，什么不值得？
- 记忆怎么存储（文件/数据库/向量）？
- 怎么在新对话开始时检索相关记忆？
- 记忆过时了怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent | MemPalace | AgentScope |
|------|------------|-------------|----------|-------------|-----------|------------|
| 核心设计 | 在上下文压缩之外引入 Session Memory File（L3 知识载体），将长期稳定信息剥离为可落盘、可审阅的 Markdown 工作笔记，由后台 subagent 异步提取，不阻塞主会话 | 忠实移植 Claude Code 记忆模式：以 `~/.openharness/data/memory/{project-name}-{sha1}/` 为存储根，`MEMORY.md` 作为索引入口，prompt 构建时注入全文并通过纯词法匹配（约 20 行）选取最相关的最多 5 个专题文件注入 | 结构化 JSON 存储（`memory.json`）分三段：user（工作上下文）、history（近远期背景）、facts（带置信度的事实数组）；`MemoryMiddleware` 在 `after_agent` 钩子异步去抖更新；内置纠错/强化信号——检测"这不对"/"对"等词语自动调整事实置信度 | 双层架构：内置层（`MEMORY.md` + `USER.md`，始终存在）+ 外部 provider 层（8 个可插拔 provider，一次只激活一个）；session 启动时烘焙冻结快照注入系统提示，中途写入不更新缓存；Nudge 机制每隔 N 轮 fork 独立后台 review agent 自主写入记忆 | 完全 raw verbatim 存储（"No summaries. Ever."）；Palace 空间隐喻（Wing → Hall → Room → Drawer）；**4 个独立层**：L0 identity（~100 tokens，始终加载，用户手写身份文本）/ L1 essential story（~500-800 tokens，始终加载，按 importance 排名 top-15 drawers）/ L2 on-demand（~200-500 tokens/次，按需 metadata 过滤）/ L3 deep search（无限制，ChromaDB 语义搜索）；`wake_up()` 加载 L0+L1（合计约 600-900 tokens 基础上下文），L2/L3 按需触发；SQLite 时态知识图谱独立于向量库 | 两层正交架构：Working Memory（`MemoryBase` 接口，4 种后端：内存/Redis/SQLAlchemy/Tablestore）+ Long-Term Memory（两条技术路线：mem0 向量语义路线 / ReMe 任务-个人-工具三分路线）；两层均继承 `StateModule` 统一序列化协议，天然适配分布式部署；LTM 以工具函数 `record_to_memory` / `retrieve_from_memory` 暴露给 LLM 自主调用 |
| 关键特点 | 多维触发保护（`shouldExtractMemory()` 综合 token 规模/增量/tool call 次数/自然停顿点）；文件即记忆（人工可读、可编辑、可版本控制）；三层知识载体明确分工（消息历史/压缩快照/Session Memory） | 词法检索完全无外部依赖，无需 embedding 模型或向量数据库；文件系统存储人类可读、可直接编辑、可用 git 版本控制；首行元数据约定（160 字符描述）将检索开销降到极低 | 纠错/强化信号隐式反馈闭环（同类项目独有）；结构化 schema + 置信度排序注入，信噪比高于扁平 markdown；去抖异步更新不阻塞主循环；tiktoken 精确 token 预算控制 | Context Fencing（`<memory-context>` 标签 + "NOT new user input" 标注）防止召回内容被误识别为用户输入；注入防护扫描（11 类威胁模式：prompt injection + 凭证外泄 + 零宽字符）；FTS5 跨 session 搜索 + 并发 LLM 摘要两级召回；单 provider 约束防止工具 schema 膨胀 | LongMemEval R@5 96.6% 零 API 成本（纯 ChromaDB 语义搜索，无 LLM 参与）；19 个 MCP 工具覆盖搜索/KG/图遍历/agent diary；PALACE_PROTOCOL 自注入（status 响应中嵌入行为规范，AI wake-up 时自动学习）；时态 KG 支持 as_of 时间点查询 | Mark 标记系统：同一 memory 实例内按字符串标签做语义分区（`get_memory(mark="tool_calls")`），同一会话内多角色/多流不互相污染；Redis 实现额外维护 `marks_index`（Redis Set）避免全量 SCAN；mem0 路线三级降级写入（user → assistant → infer=False）保证内容一定落库；ReMe 路线将 LTM 拆成 Personal/Task/Tool 三个独立实例，各自专属 prompt flow；`StateModule` 序列化使 memory 状态可跨进程持久化 |
| 局限 | 记忆内容质量依赖 LLM 归纳；跨会话 memory file 不自动合并；后台提取无用户可见反馈 | 检索仅匹配标题和首行描述，正文内容对检索不可见；无后台自动提取机制，记忆写入完全依赖显式调用；无跨项目记忆共享能力 | 无向量检索（仅置信度排序，无语义相关性召回）；置信度排序非语境感知（高置信度事实不一定与当前对话相关）；无跨 agent 记忆共享 | 冻结快照导致本轮写入的记忆下次 session 才生效；单 provider 约束无法同时激活多个外部后端；内置层全量注入无语义排序，记忆条目增多信噪比下降；后台 review agent 成本和延迟在高频短对话场景不可控 | raw verbatim 无限存储增长，无自动剪枝；AAAK 压缩模式退步 12 个百分点（84.2% vs raw 96.6%）；无注入防护，任何内容可写入向量库；L2 层按 metadata 过滤不做语义搜索，room 分类粗粒度导致大量内容落入 general | Working Memory 的 `_compressed_summary` 只是存储槽位，自动摘要触发与 LLM 调用发生在 agent/context-management 层；两层记忆无自动联动，框架不提供「超过 N 轮自动压缩写 LTM」机制，开发者须手动串联；ReMe 路线对 DashScope/OpenAI 强绑定，其他 LLM provider 直接报错；RAG 模块与 memory 完全正交无融合接口；`delete_by_mark` 未在基类声明为抽象方法，某些后端静默缺失；mem0 存在版本兼容硬编码分支（`mem0ai <= 0.1.115`）|

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 文件存储（Markdown/JSON） | 记忆落盘为人类可读文件，词法检索，和代码放在一起 | 开发者工具、单项目 agent、需要 git 版本控制 | Claude Code、OpenHarness |
| 向量数据库 | 用 embedding 做语义检索，存 ChromaDB/Pinecone 等 | 大规模知识库、需要模糊匹配的客服/个人助手 | LangChain Memory、MemGPT |
| 结构化数据库（SQLite/Postgres） | schema 化存储，精确查询，强事务保证 | 用户画像、任务历史、有明确结构的偏好数据 | 自定义任务管理 agent |
| 混合（文件 + 向量） | Markdown 保留可读性，向量索引加速语义搜索 | 生产级长期助手，兼顾可调试性和检索质量 | 需自建 |
| 双层（内置始终在线 + 外部可插拔） | 内置层保底可靠性，外部层按需扩展能力（向量/图谱/用户建模），二者独立维护 | 需要多种检索后端灵活切换、又不想每个部署都自建的通用助手 | Hermes Agent |
| Raw verbatim + 向量搜索 | 完整对话原文入库，ChromaDB embedding 做语义检索；不做 LLM 提取，零信息损失，零 API 成本 | 个人助手/长期记忆、数据隐私敏感场景、希望零 API 成本运行的本地系统；对存储空间不敏感 | MemPalace |

### 场景决策指南

**如果你在做开发者工具 / CLI agent → 文件存储**
记忆和代码一起在项目目录里，天然用 git 管理，用户能看到 agent 到底记了什么，出问题直接编辑文件。OpenHarness 这条路完全无外部依赖，词法匹配够用。

**如果你在做客服机器人，历史记录数万条 → 向量数据库**
词法匹配在这个规模下会漏掉大量语义相关内容。embedding 模型质量至关重要——先评估检索准确率再上线，别只看召回率。注意：MemPalace 实测 96.6% R@5 证明原始存储可优于 LLM 提取路线（Mem0 ~85%），embedding 质量的重要性是对的，但加 LLM 提取并不一定提升质量——见下方「raw verbatim vs LLM 提取」权衡。

**如果你在做任务管理 / 用户偏好系统 → 结构化数据库**
记忆本身有明确 schema（用户 ID、任务状态、时间戳），SQL 的精确查询和 ACID 保证比语义搜索更有价值。

**如果你在做长期个人助手，需要长达数月的记忆 → 混合方案**
Markdown 保留人类可读性和可干预性，向量索引解决规模化检索问题。代价是需要维护两套状态的一致性。

**如果你需要可换的记忆后端，又关注前缀缓存成本 → 双层 + 冻结快照（Hermes Agent 模式）**
内置层提供零依赖的基础记忆，外部 provider 通过统一 ABC 热插拔（Honcho/Hindsight/Mem0/Holographic 等 8 种）。冻结快照确保系统提示前缀每个 session 只变一次，Anthropic Prompt Cache 命中率最大化。注意：如果你的对话是长 session 且中途经常获得重要新信息，冻结快照的"下次才生效"设计会让 agent 在当前 session 内无法利用刚学到的内容。

**如果你在做 multi-agent 系统，需要在同一会话内对不同 agent/角色隔离上下文 → Mark 标记系统（AgentScope 模式）**

大多数记忆系统只有「全量历史」或「按会话隔离」两档。AgentScope 的 mark 系统提供了第三档：同一 memory 实例内的语义分区——给工具调用历史打 `tool_calls` mark，给规划步骤打 `planning` mark，不同 agent 调用 `get_memory(mark=...)` 精确取出所需子集，而不是每次传入完整历史（代码证据：`InMemoryMemory` 内部存 `list[tuple[Msg, list[str]]]`，`RedisMemory` 额外维护 `marks_index` Set 避免全量扫描）。代价是 mark 体系需要开发者自行设计和维护，mark 名称无类型约束，多人开发时容易命名漂移。与 Hermes Agent 的「单 provider 约束」不同，这里的隔离是在同一后端内做细粒度分区，而非切换后端。

**如果你需要长期记忆但诉求各不相同（个人偏好 / 任务经验 / 工具使用习惯）→ 记忆三分法（AgentScope ReMe 路线）**

将 LTM 拆成三个独立实例（`ReMePersonalLongTermMemory` / `ReMeTaskLongTermMemory` / `ReMeToolLongTermMemory`），每个子类有专属的 prompt flow 和 `record_to_memory` docstring 直接作为 LLM 工具的 description——这承认了不同类型记忆的检索策略本质不同：个人偏好需要语义相似，任务经验需要结构化步骤检索，工具指南需要基于工具名称精确匹配。与 MemPalace 的 raw verbatim 单一存储相比，三分法的代价是需要维护三个实例、且对 DashScope/OpenAI 强绑定（其他 LLM provider 直接抛 `ValueError`）。适合：已使用 AgentScope 框架且以 DashScope/OpenAI 为主要 LLM provider、需要精细化长期记忆分类的生产场景。

**如果你在做个人长期助手，要求零云依赖且不介意存储增长 → raw verbatim + ChromaDB（MemPalace 模式）**
核心取舍：不做 LLM 提取换取零 API 成本和零信息损失。LongMemEval R@5 96.6% 证明"存原文 + 好的 embedding"在检索质量上超过大多数带 LLM 提取的系统。代价是存储量随对话线性增长（无剪枝），以及没有注入防护（任何内容可写入）。适合：数据隐私敏感、离线运行、学术/研究场景、或者你就是想让 AI 记住每一句话。如果需要精确结构化信息（用户偏好、关系图谱），MemPalace 的 KG 工具可以叠加在 verbatim 层之上。

---

**[重要权衡] raw verbatim 存储 vs LLM 提取摘要**

这是记忆系统设计中最容易走错的分叉路口。直觉上觉得「LLM 提取的摘要更精炼、更有用」，但实测数据给出了反直觉的结论：

| 路线 | 代表方案 | LongMemEval R@5 | 成本 | 信息损失 |
|------|---------|-----------------|------|---------|
| raw verbatim + 语义检索 | MemPalace | **96.6%** | 零 API 成本 | 无（存原文） |
| LLM 提取 + 向量索引 | Mem0 | ~85% | 每次写入消耗 token | 有（摘要丢失细节） |
| 混合（部分提取 + 向量） | Mastra | 94.87% | 中等 | 部分损失 |

**根本原因**：LLM 摘要在提取时无法预判未来哪条细节会被查询，必然丢失部分原文信息；而 embedding 模型足够强时，直接对原文做语义检索反而比"摘要 → 检索"的两跳路径精度更高。

**决策规则**：
- **优先事实检索精度 → raw verbatim + 语义检索（无摘要）**：当你需要 agent 精确回忆「用户说过的具体内容」时，不做提取往往比 LLM 提取更准。
- **优先长期历史压缩 → LLM 提取摘要**：当历史量极大、存储成本是硬约束、或需要将记忆压缩进有限 token 预算时，摘要方案可以接受精度损失换空间效率。
- **注意**：AAAK 有损压缩（MemPalace 自身的摘要方言）在同一数据集上退步至 84.2%，再次验证了摘要路线的代价——即使是专门设计的压缩格式也无法弥补信息损失。

---

### 常见陷阱

- **什么都往记忆里塞**：检索噪音随记忆量线性增长，不加筛选的记忆比不记忆更危险。记忆写入前必须有过滤策略（Claude Code 的 `shouldExtractMemory()` 是个好例子）。
- **没有过期机制**：三个月前记录的用户偏好可能已经过时，过时记忆会系统性地误导 agent。至少要标注时间戳，定期审查高龄记忆。
- **把向量搜索当银弹**：embedding 质量差时，语义相近但实际无关的内容会被错误召回。上线前必须用真实查询做检索质量评估，不能假设 embedding 模型自动解决一切。
- **记忆写入没有反馈**：Claude Code 的后台提取在失败时用户无感知。生产系统需要监控记忆提取成功率，否则故障会悄悄积累。
- **记忆内容是注入攻击的高价值目标**：记忆最终会注入系统提示，一旦被植入恶意指令后果极严重。Hermes Agent 的 `_scan_memory_content()` 在任何写入前扫描 11 类威胁模式（prompt injection 短语、curl/wget 带 secrets 外泄模式、零宽字符）。自建记忆系统必须有等效的写前校验，不能假设 agent 只会记"正常内容"。
- **后台 review agent 的成本失控**：Nudge 机制（Hermes Agent）fork 独立 AIAgent 后台审查记忆是优雅的设计，但在高频短对话场景（如群聊 gateway）nudge_interval 设小了会大量并发 review agent，token 消耗难以预估。上线前需根据预期对话频率仔细调参，并监控 review agent 的调用量。
- **多 provider 幻觉**：允许同时激活多个记忆后端看似功能丰富，实际上会导致工具 schema 膨胀、LLM 在记忆工具选择上混乱，还要处理多后端数据一致性。Hermes Agent 用单 provider 约束（`_has_external` flag）明确拒绝第二个外部 provider，是工程上的务实取舍。
- **raw verbatim 的存储膨胀**：不做提取 = 不丢失信息，但也意味着每条对话都占存储。MemPalace 的 ChromaDB palace 目录会随使用时间线性增长，没有自动过期或剪枝。长期运行后 L1 的 MAX_DRAWERS=15 硬截断会越来越遗漏重要历史——应周期性手动归档（`mempalace_delete_drawer`）或自建剪枝脚本。
- **不提取不代表不失真**：raw verbatim 存储仍然存在分块策略引入的失真。MemPalace 的 `_chunk_by_exchange()` 只保留 AI 响应前 8 行，长响应后半段永久丢失。这违反了"No summaries. Ever."的哲学——实际上是在分块层做了隐性截断。上线前务必评估你的对话长度分布，必要时调大行数限制或改用段落分块。

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
| 无自动维护（手动删除） | raw verbatim 只增不减，MCP 提供 `mempalace_delete_drawer` 手动删除；KG 的 `invalidate()` 标记事实失效而不删除，保留历史 | MemPalace | 存储空间充裕、数据完整性优先、接受人工定期维护的场景 |
| 无自动维护 + 滑动 TTL 过期 | Redis 后端支持 `key_ttl` 参数，每次读写后调用 `_refresh_session_ttl` 刷新全部 session 相关 Key 的 TTL（滑动窗口，非固定过期）；两层记忆均无自动剪枝，但 Working Memory 可通过 TTL 实现自然淘汰 | AgentScope RedisMemory | 需要 Redis 作为 session 存储、对话稀疏（希望闲置会话自动清理）、但不需要跨 session 持久化的场景 |

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
- [[memory-system--mempalace]]
- [[memory-system--agentscope]]
