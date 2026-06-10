---
title: Context Management
aliases: [上下文管理, context window, token budgeting]
category: L1
created: 2026-04-06
updated: 2026-06-09
relations:
  - target: "[[prompt-system]]"
    type: feeds
  - target: "[[memory-system]]"
    type: uses
  - target: "[[tool-system]]"
    type: depends_on
    evidence: "AgentScope 2.x 的 `_split_tool_result_for_compression()` 和 context compression 会按工具结果大小、workspace offload 与 state.context 保留边界协同工作；工具输出策略直接决定 context 压力和 evidence 回链"
  - target: "[[runtime-state]]"
    type: uses
    evidence: "跨压缩迭代摘要、压缩状态和 session 切换边界都依赖 runtime state 正确隔离"
  - target: "[[session-recovery]]"
    type: supports
    evidence: "压缩摘要 `AgentState.summary`、未压缩消息 `AgentState.context` 和 workspace offload 引用需要随 session state 恢复，否则恢复后上下文投影不完整"
  - target: "[[evaluation-observability]]"
    type: depends_on
    evidence: "RAG / 压缩 / 截断策略上线前必须用检索质量、token、失败率等指标评估"
sources: [claude-code, openharness, deer-flow, hermes-agent, agentscope, tencentdb-agent-memory]
---

## 一句话定义

管理有限的 context window — 什么放进去、什么压缩、什么丢掉。

## 核心问题

- 当对话超长时，怎么决定保留哪些信息？
- 压缩策略：截断 vs 摘要 vs 向量检索？
- Token 预算怎么分配给不同模块（系统指令/历史/工具结果）？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent | AgentScope |
|------|------------|-------------|----------|--------------|------------|
| 核心设计 | 主动调度器而非被动救火：持续监控 token 使用、提前保留 headroom、阈值触发时执行压缩，输出可继续推理和工具调用的完整对话快照（context projection） | 极简设计：token 估算用字符数/4 的启发式公式，压缩逻辑仅 58 行，通过滑动窗口保留最近 N 条消息、将旧消息替换为单条拼接文本摘要；无自动触发，无调用模型生成摘要 | `SummarizationMiddleware` 实现自动压缩，触发条件三选一（token 数/消息数/占最大上下文比例），触发后保留最近 N 条消息，旧消息替换为摘要；tiktoken 精确计数；中间件在 `after_model` 钩子挂载，与对话循环完全解耦 | 双流水线架构：`ContextCompressor`（对话时在线压缩）+ `TrajectoryCompressor`（训练数据离线批处理），共享"头尾保护 + 中间摘要替换"核心思路；在线压缩通过 7 段结构化模板（Goal/Progress/Decisions/Files/Next Steps 等）和跨压缩迭代摘要更新（`_previous_summary` 携带前次结果），实现多次压缩后信息不清零的"信息密度蒸馏" | `AgentState.summary/context` + `Agent.compress_context()` + workspace/offloader：进入 reasoning 前按 `trigger_ratio * model.context_size` 压缩旧 context，结构化生成 summary；大工具结果由 `tool_result_limit` 和 offload 单独控制 |
| 关键特点 | `getEffectiveContextWindowSize()` 提前扣除输出预留空间；连续失败熔断机制（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）；模式感知——session_memory 模式下主动抑制自动压缩 | 整个压缩子系统 58 行完成，零外部依赖（无 tiktoken、无异步初始化）；成本追踪与压缩逻辑完全解耦；字符数估算足以支撑粗粒度判断，避免过度工程化 | 三模式触发最灵活（token_count/message_count/fraction，覆盖不同部署场景）；摘要模型可独立配置（主模型强推理，摘要用小模型降成本）；中间件模式干净解耦，替换或关闭不影响其他组件 | token-budget 尾部保护（按 token 量而非消息条数，随模型窗口自动缩放）；`_sanitize_tool_pairs()` 在每次压缩后修复孤儿 tool result/call；tools schema 纳入 token 估算（50+ 工具额外 20-30K token）；上下文探测从 API 错误实时解析真实限制并持久化缓存（`model@base_url` key），后续会话零探测复用 | `SummarySchema` 五段结构化摘要；`ChatModelBase.count_tokens()` 默认粗估但可由模型子类覆盖；压缩后的原始 messages 可通过 `offloader.offload_context()` 落到 workspace 并在 summary 中保留路径；context 压缩会清理不再保留的 read-file cache |
| 局限 | 压缩质量依赖 LLM 能力；熔断后无降级策略（无截断最旧消息等回退手段）；触发阈值为静态配置不可动态调整 | 字符数/4 对中文、代码误差可达 2-5 倍；压缩不自动触发，需 agent loop 手动检测阈值；`compact_messages()` 只做文本拼接而非语义摘要，旧上下文可读性差 | 依赖 LangChain 内置实现，无法精细控制摘要提示词；无熔断机制（摘要调用失败时无降级处理）；压缩不感知语义边界，可能在工具调用链中间截断 | 摘要失败冷却期（600 秒）内中间段被静默删除而非保留（与 Claude Code 熔断保留内容不同，风险更高）；对话时压缩仍用 4 chars/token 粗估，结构化工具输出误差可超 30%；`should_compress()` 基于上一轮返回的 prompt_tokens，本轮超大工具输出可能在下轮才触发压缩 | 默认 token 估算是 bytes/4 粗估；压缩在 reasoning 前触发，工具执行中途不会再次全局压缩；summary 覆盖式更新，没有摘要版本历史；offload 依赖 workspace，可用性和权限边界要由部署层保证；offloaded evidence 没有内置语义索引 |

### 新增源补充：TencentDB Agent Memory

| 维度 | TencentDB Agent Memory |
|---|---|
| 核心设计 | Context offload 把大工具结果转为 `refs/*.md` evidence、L1 摘要、L2 Mermaid/MMD 符号任务图；上下文压力较高时再由 L3 做 mild/aggressive/emergency 替换或删除 |
| 关键特点 | MMD 注入为 `<current_task_context>`，给 LLM 当前任务拓扑、doing/done/blocked 节点和文件线索；history MMD 有 token ratio full/meta/skip 降级；插入逻辑保护 assistant tool_use 与 tool_result 配对 |
| 局限 | active MMD 注入点只计算和 trace budget，不硬截断；`node_mapping` 覆盖依赖 prompt 和 fallback；MMD 作为 user-role 文本插入，严格 context protocol 下应迁移到 runtime/context section |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 方案 A：大 Context 硬塞 | 依赖模型大 context window，不做任何压缩，全部塞进去 | 短对话（< 10 轮）、原型验证、可接受高成本的场景 | 简单 chatbot、快速原型 |
| 方案 B：主动压缩（Compaction） | 接近上限时，让模型把旧消息压缩为摘要，替换原消息 | 长编程会话、迭代式工作、需要控制成本的长任务 | Claude Code |
| 方案 B2：迭代摘要压缩 | 压缩时复用前次摘要作为前置上下文，增量更新而非从零生成；通过结构化模板（Goal/Progress/Decisions/Files）确保多次压缩后关键信息不清零 | 超长多轮 agent 会话、需要跨多次压缩保持任务状态的场景 | Hermes Agent |
| 方案 C：RAG 检索增强 | 消息存到外部向量库，每轮检索最相关的内容注入 context | 跨会话记忆、知识库问答、客服系统 | 各类 RAG 框架、mem0 |
| 方案 D：混合策略 | 近期消息保留在 context，历史消息通过 RAG 按需检索 | 生产级长期 agent、需要长期记忆的助手 | 高级 agent 框架 |
| 方案 E：符号地图 + evidence offload | 大 payload 不进 prompt，转为 evidence refs；prompt 注入任务地图，细节由 recall/read 工具展开 | 工具结果密集、文件读取密集、需要长期保持任务方向感的 agent | TencentDB Agent Memory |

### 场景决策指南

**如果你在做短对话或快速原型（< 10 轮）→ 选方案 A（硬塞）**
- 原因：实现零成本，不引入任何复杂性，短对话根本不会触及上限
- 注意：一旦对话变长就要重新评估——200K context 的费用是 4K 的约 50 倍，别让"先跑通再说"变成线上高账单的借口；如果是给用户用的产品，要加会话长度上限保护

**如果你在做长编程会话或迭代式工作任务 → 选方案 B（主动压缩）**
- 原因：编程 agent 的对话往往夹杂大量工具调用结果（文件内容、命令输出），这些很快就吃满 context，但真正重要的是"当前任务状态"而不是每一行历史输出；压缩可以把 50K tokens 压到 5K，成本降 90%
- 注意：压缩本身有 token 成本（一次压缩约消耗 2000-5000 tokens），不能压缩太频繁；压缩是有损操作，摘要可能丢失细节——触发阈值不要设太低（建议 70-80% 窗口用量时才触发），让模型积累足够多的"值得压缩"的内容再压

**如果你在做客服、知识库问答或需要跨会话记忆 → 选方案 C（RAG）**
- 原因：用户问题往往只和历史中的少数几条消息相关，把全部历史都放进 context 是浪费；RAG 可以精准检索相关片段，还能跨会话查过去几周的记录
- 注意：检索质量是生死线——检索到不相关内容注入 context 比没有检索更糟糕（会误导模型）；embedding 模型选择和分块策略对质量影响巨大，需要专门评估；会引入额外延迟（100-300ms 检索时间）

**如果你的 agent 会话极长、需要跨多次压缩保持任务状态连贯 → 选方案 B2（迭代摘要）**
- 原因：普通压缩（方案 B）每次都从零生成摘要，多次压缩后早期的目标、决策、关键文件路径很容易从摘要中消失；迭代摘要把前次摘要作为"知识积累底座"传给 LLM，新摘要只需更新 Progress 状态，Goal 和 Decisions 自动继承——解决了"越压越忘"问题
- 注意：迭代摘要依赖 `_previous_summary` 状态在 agent 运行期间持久存在，会话切换时必须显式清除（Hermes 用 `reset_session_state()`），否则会出现前一个任务的目标污染新任务的摘要；每次压缩时对 `_previous_summary` 的处理要纳入会话管理逻辑，不能只管压缩不管状态

**如果你需要同时支持本地代理和 Web/distributed workspace → 使用 summary + evidence offload（AgentScope 2.x 模式）**

- 原因：代码代理的 context 压力常来自文件读取和工具输出。AgentScope 2.x 把 active context 留在 `AgentState.context`，把旧 context 压成 `AgentState.summary`，同时可把原文 offload 到 workspace 路径。这样本地形态可以用文件回链，Web 形态可以替换为 object store / sandbox workspace。
- 代码证据：`src/agentscope/agent/_agent.py` 的 `compress_context()` / `_compress_context_impl()`，`src/agentscope/workspace/_offload_protocol.py` 的 `offload_context()`。
- 注意：offload 不是长期记忆。它缺少检索排序、过期、事实冲突解决和 evidence index；如果要跨会话 recall，应接入 TencentDB Agent Memory / MemPalace 这类 memory 系统。

**如果你在做生产级长期运行的 agent → 选方案 D（混合策略）**
- 原因：最近 N 轮保持连贯对话上下文，历史通过 RAG 按需检索，兼顾连贯性和长期记忆
- 注意：这是最复杂的方案——要维护两套系统（context 窗口管理 + 向量库），需要设计"什么时候把 context 里的消息转移到向量库"的策略；不要一开始就上混合，先用方案 B 或 C，等真正遇到瓶颈再升级

**如果你的瓶颈主要来自工具结果而不是自然对话 → 选方案 E（TencentDB Agent Memory 模式）**
- 原因：工具结果、文件读取、搜索输出最容易撑爆 context，但这些内容往往只需要保留证据 handle 和任务拓扑。符号地图让 LLM 先知道“现在在哪个节点”，再按需读取 evidence，比盲目全量保留或纯摘要更适合长任务代码代理。
- 注意：这种方案会把 token 压力转移成 recall 工具调用压力，必须有 per-turn recall budget；地图不是事实本身，关键回答要能回到 evidence 校验。

### 常见陷阱

- **以为大 context 就不用管**：128K/200K 的 context window 给人一种"随便塞"的错觉，但成本是线性的（甚至超线性），且注意力机制在超长 context 下对中间内容关注度显著下降——"Lost in the Middle"问题真实存在
- **压缩触发太激进**：每隔 10 轮就压缩，摘要质量差，关键细节丢失，agent 开始犯"明明说过的事忘了"的错误——比没有压缩更糟糕
- **RAG 检索质量差就上线**：检索返回不相关的历史片段，模型把错误信息当成上下文事实，输出质量急剧下降；RAG 系统必须有离线评估指标（Recall@K、MRR）才能上生产
- **压缩后不验证结构完整性**：压缩输出必须是合法的对话格式（role/content 结构），自由文本摘要无法继续驱动工具调用，agent 会在压缩后的第一轮就崩溃
- **没有熔断机制**：压缩失败后不停重试，每次重试又消耗更多 token，最终雪崩——必须有最大重试次数和失败后的降级策略（如截断最旧消息）

  > **失败模式对比**：
  > - **Claude Code**：压缩失败时保留原始消息，不压缩（内容安全，但上下文持续增长）
  > - **Hermes Agent**：压缩失败后进入 600s 冷却期，冷却期内**静默删除**中间段消息（内容丢失风险更高）
  >
  > **陷阱**：Hermes 的「有冷却期」看似比 Claude Code 的「无限重试」更健壮，但实际静默删除比保留内容的危害更大。生产系统应优先确保内容不丢失，再考虑上下文长度。
- **token 估算遗漏 tools schema**：只统计消息的 token 忘了把工具 schema 一起算进去——50+ 工具的 agent（如 Hermes）中 schema 序列化额外消耗 20-30K token，导致实际压缩阈值比预期晚触发，第一次 API 调用就因超限报错
- **压缩后不做工具对完整性修复**：压缩切割中间段后，极易产生两类孤儿：tool result 引用了已被压缩掉的 call_id（API 报 400）、assistant 的 tool_calls 丢失了对应 result（模型陷入困惑状态）；压缩完成后必须扫描修复，否则第一轮工具调用就会崩溃
- **符号地图没有质量指标**：MMD/graph 看起来结构化，不代表事实都正确归因。需要统计 node 覆盖率、fallback/skip 率、错误归因和额外 recall tool call 次数，否则错误地图会稳定误导主 LLM。
- **只用 prompt 约束预算**：把“最多搜索 3 次”写进 guide 有帮助，但不是 hard limit。生产实现应在 tool/hook 层按 turn 计数并拒绝超额调用。
- **上下文窗口大小写死或查表不准**：模型配置的最大 context 长度可能与实际 API 端点限制不同（尤其是多租户服务、不同订阅档位），写死或查本地表格都会误判压缩时机；正确做法是从 API 错误信息中实时解析实际限制并持久化缓存，后续会话直接复用
- **默认 token 粗估当作精确预算**：AgentScope 2.x 的 base `count_tokens()` 是 bytes/4 启发式估算，provider 子类若不覆写，压缩触发可能偏移。生产系统要么使用 provider 精确计数，要么把阈值留出更大 safety margin。
- **offload 后没有 evidence index**：把旧 context 写到 workspace 只解决“原文还在”，不解决“什么时候该召回哪段”。长任务需要维护 evidence URI、摘要、节点归因和召回预算。

## 相关模式

- [[symbolic-context-map-progressive-recall]] — evidence-backed 符号地图 + 渐进召回

## L2 详情

- [[context-management--claude-code]]
- [[context-management--openharness]]
- [[context-management--deer-flow]]
- [[context-management--hermes-agent]]
- [[context-management--agentscope]]
