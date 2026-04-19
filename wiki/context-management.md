---
title: Context Management
aliases: [上下文管理, context window, token budgeting]
category: L1
created: 2026-04-06
updated: 2026-04-15
relations:
  - target: "[[prompt-system]]"
    type: feeds
  - target: "[[memory-system]]"
    type: uses
  - target: "[[tool-system]]"
    type: depends_on
    evidence: "AgentScope TruncatedFormatterBase._truncate() 和 _compress_memory_if_needed() 均用 tool_call_ids 集合追踪 tool_use/result 配对，截断和压缩边界必须感知 tool 边界才能产出合法 API 消息序列；工具调用密度直接影响 keep_recent 的实际 token 保留量。来源：agentscope/formatter/_truncated_formatter_base.py, agentscope/agent/_react_agent.py"
sources: [claude-code, openharness, deer-flow, hermes-agent, agentscope]
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
| 核心设计 | 主动调度器而非被动救火：持续监控 token 使用、提前保留 headroom、阈值触发时执行压缩，输出可继续推理和工具调用的完整对话快照（context projection） | 极简设计：token 估算用字符数/4 的启发式公式，压缩逻辑仅 58 行，通过滑动窗口保留最近 N 条消息、将旧消息替换为单条拼接文本摘要；无自动触发，无调用模型生成摘要 | `SummarizationMiddleware` 实现自动压缩，触发条件三选一（token 数/消息数/占最大上下文比例），触发后保留最近 N 条消息，旧消息替换为摘要；tiktoken 精确计数；中间件在 `after_model` 钩子挂载，与对话循环完全解耦 | 双流水线架构：`ContextCompressor`（对话时在线压缩）+ `TrajectoryCompressor`（训练数据离线批处理），共享"头尾保护 + 中间摘要替换"核心思路；在线压缩通过 7 段结构化模板（Goal/Progress/Decisions/Files/Next Steps 等）和跨压缩迭代摘要更新（`_previous_summary` 携带前次结果），实现多次压缩后信息不清零的"信息密度蒸馏" | 三层正交架构：**Token 计数层**（5 种后端统一 async 接口，本地或 API 调用均可）、**Formatter 截断层**（format 时按 token 限制自动 drop 最旧完整 turn）、**Memory 压缩层**（agent 级 LLM 驱动，结构化 SummarySchema 输出，压缩后旧消息打 COMPRESSED 标记而非删除）。三层完全解耦，可按需任意组合。 |
| 关键特点 | `getEffectiveContextWindowSize()` 提前扣除输出预留空间；连续失败熔断机制（`MAX_CONSECUTIVE_AUTOCOMPACT_FAILURES`）；模式感知——session_memory 模式下主动抑制自动压缩 | 整个压缩子系统 58 行完成，零外部依赖（无 tiktoken、无异步初始化）；成本追踪与压缩逻辑完全解耦；字符数估算足以支撑粗粒度判断，避免过度工程化 | 三模式触发最灵活（token_count/message_count/fraction，覆盖不同部署场景）；摘要模型可独立配置（主模型强推理，摘要用小模型降成本）；中间件模式干净解耦，替换或关闭不影响其他组件 | token-budget 尾部保护（按 token 量而非消息条数，随模型窗口自动缩放）；`_sanitize_tool_pairs()` 在每次压缩后修复孤儿 tool result/call；tools schema 纳入 token 估算（50+ 工具额外 20-30K token）；上下文探测从 API 错误实时解析真实限制并持久化缓存（`model@base_url` key），后续会话零探测复用 | `CompressionConfig` 支持独立压缩模型（主模型做推理、便宜小模型做摘要，分离成本）；结构化压缩输出（SummarySchema 5 字段各有 max_length，强制 JSON，避免自由文本摘要格式混乱）；tool_use/tool_result 配对安全截断（`tool_call_ids` 集合追踪，截断和压缩边界计算共用同一套机制）；被压缩消息保留原始存储可回溯（COMPRESSED 标记，正常检索跳过，调试/审计可按 ID 查询）；OpenAI 图片 token 精确实现（tile 算法，按模型系列差异化参数）；HuggingFace counter 直接调用 `apply_chat_template(tokenize=True, tools=tools)`，工具 schema 自动纳入计数 |
| 局限 | 压缩质量依赖 LLM 能力；熔断后无降级策略（无截断最旧消息等回退手段）；触发阈值为静态配置不可动态调整 | 字符数/4 对中文、代码误差可达 2-5 倍；压缩不自动触发，需 agent loop 手动检测阈值；`compact_messages()` 只做文本拼接而非语义摘要，旧上下文可读性差 | 依赖 LangChain 内置实现，无法精细控制摘要提示词；无熔断机制（摘要调用失败时无降级处理）；压缩不感知语义边界，可能在工具调用链中间截断 | 摘要失败冷却期（600 秒）内中间段被静默删除而非保留（与 Claude Code 熔断保留内容不同，风险更高）；对话时压缩仍用 4 chars/token 粗估，结构化工具输出误差可超 30%；`should_compress()` 基于上一轮返回的 prompt_tokens，本轮超大工具输出可能在下轮才触发压缩 | 截断策略单一（只有"删最旧完整 turn"，无滑动窗口或重要性评分）；压缩触发仅在每轮 reply 开始前检查一次，单轮大量并行工具调用后无法 mid-flight 再次触发；压缩不处理多模态内容（源码 TODO 未解决）；RAG 检索结果不经过 Formatter 截断路径，超大检索结果无法被框架感知；AnthropicTokenCounter/GeminiTokenCounter 需网络 API 调用增加延迟；`keep_recent` 以完整 turn 数计而非 token 数，保留量不可预测；sentence 分割只支持英文 |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 方案 A：大 Context 硬塞 | 依赖模型大 context window，不做任何压缩，全部塞进去 | 短对话（< 10 轮）、原型验证、可接受高成本的场景 | 简单 chatbot、快速原型 |
| 方案 B：主动压缩（Compaction） | 接近上限时，让模型把旧消息压缩为摘要，替换原消息 | 长编程会话、迭代式工作、需要控制成本的长任务 | Claude Code |
| 方案 B2：迭代摘要压缩 | 压缩时复用前次摘要作为前置上下文，增量更新而非从零生成；通过结构化模板（Goal/Progress/Decisions/Files）确保多次压缩后关键信息不清零 | 超长多轮 agent 会话、需要跨多次压缩保持任务状态的场景 | Hermes Agent |
| 方案 C：RAG 检索增强 | 消息存到外部向量库，每轮检索最相关的内容注入 context | 跨会话记忆、知识库问答、客服系统 | 各类 RAG 框架、mem0 |
| 方案 D：混合策略 | 近期消息保留在 context，历史消息通过 RAG 按需检索 | 生产级长期 agent、需要长期记忆的助手 | 高级 agent 框架 |

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

**如果你需要对接多家模型（OpenAI / Anthropic / Gemini / HuggingFace 混用）且需要精确 token 计数 → 使用 AgentScope 的多后端 Token Counter 方案**

- 原因：不同模型的 token 计数方式差异显著——OpenAI 有视觉 tile 算法，Anthropic 需调用官方 count_tokens API，HuggingFace 模型有各自的 chat_template；AgentScope 把 5 种计数后端统一为同一接口（`TokenCounterBase.count(messages, tools)`），切换模型只需换 counter 实例，不影响截断逻辑。
- 代码证据：`agentscope/token/` 下 5 个 counter 实现，均继承 `TokenCounterBase`，接口签名完全一致；`TruncatedFormatterBase` 构造参数为 `token_counter: TokenCounterBase`，运行时多态。
- 与已有源对比：Claude Code 和 Hermes Agent 的 token 计数深度绑定特定后端（tiktoken / API 错误解析），DeerFlow 只用 tiktoken，OpenHarness 用字符数/4；AgentScope 是目前对比源中唯一做到"多模型 token 计数后端统一抽象"的实现，尤其对混合使用大小模型的场景（主模型推理 + 小模型压缩）有直接参考价值。

**如果你需要在压缩后保留审计/调试能力，不希望物理删除旧消息 → 选标记式压缩（COMPRESSED 标记）而非物理删除**

- 原因：物理删除旧消息后，压缩质量问题（如摘要丢失关键决策）无法事后还原排查；AgentScope 的 `COMPRESSED` 标记方案使旧消息仍保留在 `InMemoryMemory` 的 `list[tuple[Msg, list[str]]]` 结构中，正常检索路径通过 `exclude_mark=COMPRESSED` 跳过，但可按 ID 直接访问原始消息。
- 代码证据：`memory.update_messages_mark(msg_ids, new_mark=_MemoryMark.COMPRESSED)`；`get_memory(exclude_mark=_MemoryMark.COMPRESSED)` 正常流程；`_compressed_summary` 作为 StateModule 状态字段持久化，跨 session 恢复后摘要不丢。
- 与已有源对比：Claude Code / DeerFlow / Hermes Agent 均用"替换"而非"标记"，旧消息被摘要文本覆盖后不可恢复；AgentScope 的标记方案在"可回溯性"维度提供了新的设计参考，适合需要合规审计或压缩质量离线评估的场景。

**如果你在做生产级长期运行的 agent → 选方案 D（混合策略）**
- 原因：最近 N 轮保持连贯对话上下文，历史通过 RAG 按需检索，兼顾连贯性和长期记忆
- 注意：这是最复杂的方案——要维护两套系统（context 窗口管理 + 向量库），需要设计"什么时候把 context 里的消息转移到向量库"的策略；不要一开始就上混合，先用方案 B 或 C，等真正遇到瓶颈再升级

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
- **上下文窗口大小写死或查表不准**：模型配置的最大 context 长度可能与实际 API 端点限制不同（尤其是多租户服务、不同订阅档位），写死或查本地表格都会误判压缩时机；正确做法是从 API 错误信息中实时解析实际限制并持久化缓存，后续会话直接复用
- **独立压缩模型与主模型 token 计数器不一致**：`CompressionConfig.agent_token_counter` 必须与 agent 主模型使用相同的 counter；若主模型是 Anthropic 但 `agent_token_counter` 用 `CharTokenCounter`，触发阈值会严重偏移。AgentScope 文档建议明确传入与主模型匹配的 counter 实例。
- **`keep_recent` 用 turn 数而非 token 数导致保留量不可预测**：`keep_recent=3` 在工具密集场景可能保留数万 token（每个 turn 有大量 tool_use/result），在纯文本对话中只保留几百 token；若需要精确的 token 预算控制，应改用 `keep_recent` 的 token-budget 版本或在压缩前先估算保留 turn 的 token 总量。

## L2 详情

- [[context-management--claude-code]]
- [[context-management--openharness]]
- [[context-management--deer-flow]]
- [[context-management--hermes-agent]]
- [[context-management--agentscope]]
