---
pattern: rag
category: orchestration
tags: [retrieval, knowledge-injection, grounding, hallucination-reduction, context-augmentation]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_wiki: [wiki/context-management, wiki/memory-system, wiki/prompt-system]
related_patterns: [prompt-chaining, structured-output, few-shot]
---

# Retrieval-Augmented Generation (RAG)

> 检索外部知识注入 prompt，让模型基于真实数据回答而非凭记忆编造，大幅减少幻觉。

## 本质

闭卷考试 vs 开卷考试。

LLM 的参数知识是训练时"背"进去的，截止日期固定，覆盖面有限，还会"记错"——也就是幻觉。RAG 把规则改成了开卷：模型回答前先去"翻书"，把相关资料找出来塞进 prompt，然后基于这些真实文字生成答案。

技术本质：把**参数化记忆**（模型权重里的隐式知识）和**非参数化记忆**（外部可检索文档）结合。前者提供语言和推理能力，后者提供事实和领域知识。两者分工，各司其职。

这个范式由 Lewis et al. (2021) 在 Meta AI 提出，原始形态是端到端可训练的 seq2seq + 稠密检索器。现在工程中更常见的是无需训练的 "Prompt-time RAG"：查询时检索 → 结果拼进 prompt → 模型生成。

## 什么时候用

**需要最新信息** — 模型知识截止日期之后的事件、实时价格、最新法规。模型的参数知识是静态快照，RAG 是唯一不重新训练就能更新知识的方式。

**领域专业知识** — 企业内部文档、私有代码库、产品手册、医疗规程。这些知识根本不在训练数据里，模型靠"猜"只会给出听起来合理但错误的答案。

**减少幻觉、需要可溯源答案** — 每条答案都能指向具体来源段落，用户可以自行核验。高风险场景（法律、医疗、财务）尤其需要。

**企业知识库问答** — 接入内部 Wiki、客服知识库、合规文档，让模型成为有根据的专家而非乱猜的通才。

**长尾知识查询** — 训练数据中出现频率低的专业知识，模型容易幻觉，RAG 能用精准检索弥补。

## 什么时候不用

**通用常识问答** — "光速是多少"、"Python 列表怎么排序"，模型直接回答更快，RAG 只会增加延迟和成本。

**创意任务** — 写故事、写诗、头脑风暴。创意需要自由发挥，强行注入检索到的"相关文档"反而会约束想象力。

**Context window 已够用的小文档** — 如果整份文档塞进 prompt 还在 context 限制内，直接塞就行，没必要建检索管道。文档 < 50k tokens 可以考虑直接注入。

**实时性不重要的简单对话** — 闲聊、情绪支持、格式转换，知识准确性不是瓶颈，RAG 没有收益。

**检索质量难以保证的冷启动阶段** — 没有高质量语料库就别用 RAG，没有知识的检索比没有检索更危险（给模型喂进去错误参考）。

## 模板

### 基础版

**系统 prompt + 检索注入结构**：

```text
你是一个知识问答助手。回答时必须基于以下检索到的参考资料，不要使用参考资料之外的知识。
如果参考资料中没有足够信息来回答问题，请直接说"根据现有资料无法回答"，不要编造内容。
每个关键陈述后请注明来源编号，如 [1]、[2]。

参考资料：
[1] {{retrieved_chunk_1}}
来源：{{source_1}}

[2] {{retrieved_chunk_2}}
来源：{{source_2}}

[3] {{retrieved_chunk_3}}
来源：{{source_3}}

问题：{{user_question}}
```

关键点：
- 明确限制模型"只用参考资料"，降低模型用自身知识覆盖检索结果的概率
- 要求注明来源编号，强化可溯源性
- 显式给出"资料不足时的退出路径"，避免模型强行编答案

---

**带置信度引导的版本**（适合高风险场景）：

```text
基于以下资料回答问题。

资料：
{{retrieved_context}}

回答格式：
1. 答案：[基于资料的直接回答]
2. 依据：[引用资料中的具体句子或段落]
3. 置信度：[高 / 中 / 低] — 判断依据：[资料与问题的匹配程度]
4. 资料局限：[资料中缺失的信息，如有]

问题：{{user_question}}
```

---

### Agent 集成版

在 agent 中把检索作为工具调用，让模型自主决定何时检索、检索什么：

```text
你是一个研究助手，可以使用以下工具：

- search_knowledge_base(query: str) → 返回最相关的 3-5 个文档片段
- get_document(doc_id: str) → 返回完整文档内容

工作流程：
1. 分析用户问题，判断需要哪类知识
2. 构造精准检索 query（不要直接用用户原话，要提炼关键词）
3. 检索后评估结果是否足够回答问题
4. 如需要，用更精细的 query 进行二次检索
5. 基于检索结果组织回答，标注来源

检索 query 构造规则：
- 用名词和关键概念，避免问句形式
- 如果问题包含多个子问题，分开检索
- 专有名词保持原样，不要改写
```

工具定义示例（Python + Claude API）：

```python
tools = [
    {
        "name": "search_knowledge_base",
        "description": "搜索知识库，返回与 query 最相关的文档片段。用于回答需要专业知识或最新信息的问题。",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索关键词，应为名词短语而非完整问句"
                },
                "top_k": {
                    "type": "integer",
                    "description": "返回的文档片段数量，默认 3，复杂问题可设 5",
                    "default": 3
                }
            },
            "required": ["query"]
        }
    }
]
```

## 变体

**Naive RAG** — 最基础形态：用户 query → 向量检索 → top-k 文档拼接进 prompt → 生成。实现简单，适合验证场景，但检索质量依赖 embedding 模型，且不做任何重排。

**Advanced RAG（重排序 + 混合检索）** — 在检索层引入 BM25（关键词）+ 向量检索混合，结果用 cross-encoder reranker 重排，取分数最高的 k 个。显著提升检索质量，尤其对专业术语和精确名词有效。代价是延迟增加（reranker 是额外推理）。

**Modular RAG** — 把 RAG 管道拆成可替换模块：路由模块（判断是否需要检索）、检索模块（多路召回）、融合模块（合并多源结果）、生成模块。每个模块独立可配置，适合企业级复杂场景。

**Agentic RAG（Agent 自主检索）** — 把检索变成 agent 的工具之一，模型自己决定何时检索、检索什么、是否需要多轮检索。可以处理多跳推理（问题 A 的答案才能引导出检索 query B）。延迟最高但能力最强，适合复杂研究类任务。

**HyDE（Hypothetical Document Embeddings）** — 先让模型生成一个"假设性答案"，用这个假设答案的向量去检索真实文档，再用真实文档重新生成答案。适合用户 query 与文档表达风格差异很大的场景（如口语问题 vs 专业文档）。

**Self-RAG** — 模型自我反思版：生成时动态决定是否需要检索（Retrieve token），检索后评估文档相关性（ISREL token），生成后评估答案是否有依据（ISSUP token）。每段输出都有质量标注。训练成本高，但能大幅减少不必要检索和幻觉生成。

## 组合与选择

**RAG + CoT** → 检索提供事实，CoT 保证推理链正确。适合需要多步推理且依赖外部知识的任务（如：检索法规条文 → CoT 推导合规结论）。注意：CoT 的推理步骤里也可能出现幻觉，RAG 只能锚定事实输入，不能保证推理过程无误。

**RAG + Structured Output** → 从检索结果中提取结构化信息，如从多份文档中抽取表格、时间线、对比数据。先 RAG 检索相关段落，再用 Structured Output 强制按 schema 输出。注意 chunk 边界问题——结构化字段可能跨 chunk 分散，需要调整检索策略（扩大 chunk 大小或做 parent-child 检索）。

**RAG + Prompt Chaining** → 多轮检索管道：第一轮检索生成初步答案 → 识别知识缺口 → 第二轮针对性检索 → 综合生成最终答案。适合复杂研究类问题，但延迟是普通 RAG 的 2-3 倍。

**RAG vs Fine-tuning** → 两者解决不同问题，不是替代关系。RAG 解决"知道哪些事实"（动态、可更新），fine-tuning 解决"用什么风格和格式回答"（静态、需要重训练）。知识密集型任务先上 RAG；如果模型的行为风格不符合要求再考虑 fine-tuning。两者可以叠加。

## 模型差异

| 模型 | 表现描述 | 注意事项 |
|------|---------|---------|
| Claude 4 | 对检索结果的引用和归因能力强，能准确识别"参考资料不够"的情况并说明；context window 大（200k），可以注入更多检索结果 | 给明确的"只用参考资料"指令效果更好；长 context 下仍存在"lost in the middle"问题，重要信息放头尾 |
| GPT-4 / GPT-4o | function calling 与检索工具集成成熟，Assistants API 内置了 file search（向量检索）；GPT-4o 的 128k context 支持较大检索窗口 | 原生 RAG 工具绑定 OpenAI 生态；自建检索管道时模型倾向于忽略检索结果偏用自身知识，需要强调指令 |
| Gemini 2 | 原生集成 Google Search grounding，可以实时检索网页；1M context window 理论上可以注入超大语料 | 超长 context 下生成质量下降，实测 RAG 效果建议控制在 200k 以内；grounding 功能目前与自定义知识库集成需要额外工程 |
| 开源模型（LLaMA 3、Qwen 2.5、Mistral） | 配合 LangChain/LlamaIndex 有大量现成 RAG 集成；本地部署可以处理敏感数据 | 指令遵循能力弱于闭源模型，"只用参考资料"的约束更容易被突破；建议加更强的 system prompt 限制 + 检索结果格式化标注 |

## 常见踩坑

**检索质量差（Garbage In, Garbage Out）** — 现象：模型基于检索结果给出错误答案，或答案和问题风马牛不相及。原因：embedding 模型语义理解不准、chunk 切割破坏了语义完整性、索引覆盖不全。避免方式：评估检索质量（独立于生成）；用 reranker 过滤低相关度结果；检索不到时宁可说"不知道"也不要喂噪声给模型。

**Context 过载（把 top-20 都塞进去）** — 现象：检索 token 数占满 context window，答案质量下降，模型开始混淆来源。原因：检索召回的文档太多、太长，超过模型有效处理范围。避免方式：控制注入的 chunk 总长（建议不超过 context window 的 40%）；用 reranker 筛选真正相关的 3-5 个；chunk 太长时做摘要压缩再注入。

**Chunk 切割不当** — 现象：关键信息被切断，检索到的片段语义不完整，模型无法从中提取有效知识。原因：按固定 token 数切割，不考虑语义边界。避免方式：按段落、章节等语义单位切割；用 parent-child 检索（小 chunk 用于检索，返回时扩展到父级大 chunk 提供上下文）；重叠切割（chunk 之间有 10-20% 重叠）。

**模型忽略检索结果偏用自身知识** — 现象：明明检索到了正确内容，模型还是给出了错误的、基于参数知识的答案。原因：模型的参数知识和指令遵循权衡问题，尤其是检索结果和模型"认为"的事实冲突时。避免方式：在 prompt 中加强"必须基于参考资料"的约束；让模型先引用原文再解释；用 faithfulness 评估指标检测这类漂移（可以用 RAGAS 框架自动评估）。

**检索 Query 质量低** — 现象：用户问了个复杂问题，检索 query 直接用原话，召回的文档风马牛不相及。原因：用户的口语表达和文档的书面表达在向量空间里距离远。避免方式：Query rewriting（让模型先把问题改写成更适合检索的形式）；HyDE（生成假设文档再检索）；多 query 检索（拆分原始问题的多个方面，分别检索后合并）。

**没有"不知道"的退出路径** — 现象：检索结果不相关，但模型强行基于无关文档生成了看起来合理的答案。原因：prompt 没有给模型"拒绝回答"的许可。避免方式：显式在 prompt 中说"如果参考资料不足以回答，请说明资料不足，不要猜测"；加检索相关性阈值（低于阈值的结果不注入，直接告知用户无法回答）。

## 来源

- Lewis, P. et al. (2021). *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*. NeurIPS 2020. https://arxiv.org/abs/2005.11401
- Gao, Y. et al. (2023). *Retrieval-Augmented Generation for Large Language Models: A Survey*. https://arxiv.org/abs/2312.10997
- Asai, A. et al. (2023). *Self-RAG: Learning to Retrieve, Generate, and Critique through Self-Reflection*. https://arxiv.org/abs/2310.11511
- Guu, K. et al. (2020). *REALM: Retrieval-Augmented Language Model Pre-Training*. ICML 2020. https://arxiv.org/abs/2002.08909
- Meta AI Blog. *Retrieval Augmented Generation: Streamlining the creation of intelligent NLP models*. https://ai.meta.com/blog/retrieval-augmented-generation-streamlining-the-creation-of-intelligent-natural-language-processing-models/
- Prompt Engineering Guide (DAIR.AI). Retrieval Augmented Generation. https://www.promptingguide.ai/techniques/rag
- 关联 wiki: [[wiki/context-management]] [[wiki/memory-system]] [[wiki/prompt-system]]
