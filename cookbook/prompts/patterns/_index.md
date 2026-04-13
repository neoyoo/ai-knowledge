# Prompt Patterns

> 核心提示词设计模式。每个 pattern 说明什么时候用、什么时候不用、怎么用、和其他 pattern 怎么组合。

## 按类别浏览

### Reasoning（推理）
- [Chain-of-Thought](chain-of-thought.md) — 引导模型在回答前展示推理步骤，将隐性思考外显化，提升复杂推理任务的准确率
- [Self-Consistency](self-consistency.md) — 对同一问题用 CoT 生成多条推理路径，取多数一致的答案，用冗余换可靠性
- [Tree of Thoughts](tree-of-thoughts.md) — 将推理过程从线性链扩展为树状搜索，探索多条路径并评估回溯，解决需要探索和规划的复杂问题
- [Generate Knowledge](generate-knowledge.md) — 先让模型生成相关知识/事实，再用这些知识回答问题——相当于"自己给自己开卷"，激活模型深层知识而无需外部检索
- [PAL](pal.md) — 让模型把推理过程写成代码而非自然语言，然后执行代码得到精确结果，彻底避免计算错误
- [Directional Stimulus](directional-stimulus.md) — 在 prompt 中加入定向提示（hint/keyword），引导模型朝特定方向推理，而不是让模型自由发挥
- [Graph Prompting](graph-prompting.md) — 用图结构（节点 + 边）组织 prompt 中的信息和推理路径，适合处理实体关系和网络结构问题
- [Multimodal CoT](multimodal-cot.md) — 将 CoT 推理扩展到多模态输入（文本 + 图像），让模型基于视觉信息进行分步推理，而不只是直接看图答题

### Action（行动）
- [ReAct](react.md) — 让模型在推理（Thought）和行动（Action）之间交替循环，根据观察结果动态调整下一步，是大多数 Agent 的核心运行模式
- [Reflexion](reflexion.md) — ReAct 的升级版——在行动循环中加入自我反思层，让模型从失败中提炼教训，带着教训重试，而不是盲目重来
- [ART](art.md) — 自动从任务库中检索相关示例，生成包含推理步骤和工具调用的解决方案，减少手动 prompt 工程

### Output（输出控制）
- [Structured Output](structured-output.md) — 约束模型输出为可预测的结构化格式（JSON/XML/Schema），让程序能可靠解析 LLM 回复，是 tool calling 和数据管道的基础
- [Few-shot Prompting](few-shot.md) — 在 prompt 中给出示例，通过"举一反三"引导模型学会任务模式和输出格式，无需微调就能让模型掌握新任务

### Orchestration（编排）
- [Prompt Chaining](prompt-chaining.md) — 将复杂任务拆成多个简单 prompt 的流水线，前一步的输出作为后一步的输入，逐步完成目标
- [RAG](rag.md) — 检索外部知识注入 prompt，让模型基于真实数据回答而非凭记忆编造，大幅减少幻觉

### Meta（元设计）
- [System Prompt Design](system-prompt-design.md) — 系统提示词定义 LLM 在本次交互中的身份、能力、限制和行为规范，是 agent 设计的第一步也是最重要的一步
- [Meta Prompting](meta-prompting.md) — 用 LLM 来生成、改进或优化 prompt 本身，让模型成为 prompt 工程师
- [Zero-shot Prompting](zero-shot.md) — 不给任何示例，直接用指令让模型完成任务。是最基础也最常用的 prompting 方式，也是所有复杂技巧的起点
- [Active-Prompt](active-prompt.md) — 根据模型的不确定性主动选择最有价值的示例加入 prompt，而非随机或人工选择
- [APE](ape.md) — 用 LLM 自动搜索和优化 prompt 指令，找到比人类手写更优的 prompt 表达方式

## 怎么选

- 需要模型推理 → Chain-of-Thought
- 推理结果不稳定、要求高准确率 → Self-Consistency（多路采样投票）
- 问题需要探索和回溯 → Tree of Thoughts
- 需要模型调用工具 → ReAct + Structured Output
- 模型做完一件事失败后要自我改进 → Reflexion
- 有精确计算需求（数学/日期） → PAL
- 任务太复杂一个 prompt 搞不定 → Prompt Chaining
- 需要注入外部知识 → RAG
- 模型知识够用但激活不充分 → Generate Knowledge
- 搭一个新 agent → System Prompt Design（第一步）
- 需要稳定输出格式 → Structured Output + Few-shot
- 需要自动优化 prompt → APE / Meta Prompting
