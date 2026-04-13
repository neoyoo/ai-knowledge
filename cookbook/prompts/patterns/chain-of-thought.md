---
pattern: chain-of-thought
category: reasoning
tags: [reasoning, step-by-step, math, logic, agent-core]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: beginner
related_wiki: [wiki/prompt-system, wiki/query-loop]
related_patterns: [react, prompt-chaining, structured-output, self-consistency]
---

# Chain-of-Thought (CoT)

> 引导模型在回答前展示推理步骤，将隐性思考外显化，提升复杂推理任务的准确率。

## 本质

想象让一个学生解数学题：如果只问"答案是什么"，他可能凭感觉猜；但要求他写出每一步解题过程，错误就无处隐藏，他自己也会发现问题。CoT 对 LLM 做的是同一件事——强制模型把推理过程写出来。

写出步骤本身会让推理质量变好，因为每一步的输出成为下一步的输入上下文。模型不再是在一次 forward pass 里"猜答案"，而是在 token 序列里一步步构建逻辑。这个能力在大模型（约 100B+ 参数）中作为涌现能力出现，小模型效果有限。

## 什么时候用

**数学与定量推理** — 多步计算、代数、概率。模型需要中间结果才能得出正确答案。

**逻辑与条件判断** — 如果/那么链式推断、规则匹配、真值表评估。步骤越多，CoT 收益越大。

**复杂代码调试** — 要求模型逐行分析代码执行流程，定位 bug 根因而非直接猜测。

**Agent 开发场景** — Tool selection（为什么选这个工具而不是那个）、Plan 分解（任务拆成哪些子步骤）、Error recovery（出了什么错、下一步怎么做）。CoT 让 agent 的决策链路可观测、可调试。

**需要可审查性的场景** — 医疗、法律、金融等高风险领域，推理过程本身就是输出的一部分，需要人工审核。

## 什么时候不用

**简单事实查询** — "法国的首都是哪里"不需要推理步骤，加 CoT 只是浪费 token，且可能引入不必要的犹豫。

**创意写作和头脑风暴** — 逐步推理会压制联想和创造力，让输出变得机械。需要发散思维时反其道而行。

**延迟敏感的实时应用** — CoT 让输出更长，TTFT（time to first token）和总延迟都会上升。流式场景或快速响应产品要权衡。

**分类/打标签等结构化输出** — 如果任务只是"这句话是正面还是负面"，强制推理步骤会干扰结构化输出的格式控制，不如直接用 Structured Output pattern。

## 模板

### 基础版

**Zero-shot CoT** — 在 prompt 末尾加一句魔法咒语，触发模型自发推理：

```text
{{problem_statement}}

Let's think step by step.
```

示例：

```text
我去市场买了 10 个苹果，给了邻居 2 个，给了修理工 2 个，又买了 5 个，最后吃了 1 个。
我还剩多少个苹果？

Let's think step by step.
```

模型输出会包含完整推理链，最终给出正确答案（10 个）。

---

**Few-shot CoT** — 提供带推理步骤的示例，教会模型你期望的推理格式：

```text
Q: 果园里有 15 棵树。工人今天种了一些树，种完后共有 21 棵。他们今天种了几棵？
A: 开始有 15 棵，最后有 21 棵，差值就是新种的数量。21 - 15 = 6 棵。答案是 6。

Q: 停车场有 3 辆车，又来了 2 辆，现在有几辆？
A: 原来 3 辆，来了 2 辆，3 + 2 = 5 辆。答案是 5。

Q: {{new_question}}
A:
```

关键点：示例中的推理步骤格式要和你期望的输出格式一致，模型会模仿。

---

### Agent 集成版

在 system prompt 中嵌入 CoT 指令，让推理行为成为 agent 的默认工作模式：

```text
你是一个任务执行 agent。在每次行动前，你必须用 <thinking> 标签展示你的推理过程。

推理格式：
<thinking>
1. 当前状态：[描述你对当前情况的理解]
2. 目标分析：[这一步需要达成什么]
3. 选项评估：[有哪些可选工具或行动，各自的优劣]
4. 决策：[我选择 X，因为 Y]
</thinking>

[然后执行你的行动或给出回答]

在以下情况下，推理步骤不可省略：
- 选择使用哪个工具之前
- 遇到错误或意外结果后
- 需要分解复杂任务时
```

Tool selection 前的推理指令（更轻量的版本）：

```text
在调用任何工具前，先用一句话说明你为什么选择这个工具而不是其他选项。
格式：[理由] → [工具调用]
```

Error recovery 推理模板：

```text
执行失败时，按以下格式分析：

<error_analysis>
- 发生了什么：{{error_description}}
- 可能原因：[列出 2-3 个假设]
- 最可能的原因：[选一个，说明理由]
- 恢复策略：[具体的下一步行动]
</error_analysis>
```

### 变体

**Zero-shot CoT** — 无需示例，在 prompt 末尾加 "Let's think step by step" 即可触发。适用于：示例难以构造、问题类型多变的通用场景。来自 Kojima et al. (2022)。

**Few-shot CoT** — 提供 3-8 个包含完整推理链的示例。适用于：特定领域有固定推理格式要求，或 zero-shot 效果不稳定时。来自 Wei et al. (2022)。

**Auto-CoT** — 用 LLM 自动生成 few-shot 示例中的推理链，而非手工编写。流程：对数据集问题聚类 → 每类抽代表性问题 → zero-shot CoT 生成推理链 → 组装为 few-shot prompt。适用于：大规模场景、手工构造 few-shot 成本高时。来自 Zhang et al. (2022)。

**Self-Consistency** — 对同一问题采样多条推理路径（temperature > 0），取多数答案作为最终输出。不改变单次 prompt，通过冗余推理提升鲁棒性。适用于：高精度要求、答案可验证的场景；代价是推理成本乘以采样次数。详见将来的独立页面 [[cookbook/prompts/patterns/self-consistency]]。来自 Wang et al. (2022)。

## 组合与选择

**CoT + ReAct** → 推理 + 工具调用交织。CoT 负责"想清楚"，ReAct 负责"拿信息再想"。适用于需要外部信息才能完成推理的 agent 任务（搜索、数据库查询等）。CoT 单独使用时模型只能基于已知知识推理，加了 ReAct 才能扩展到实时信息。

**CoT + Structured Output** → 先 CoT 推理，最后用 JSON/特定格式输出结论。CoT 保证推理质量，Structured Output 保证格式稳定。实现方式：在 `<thinking>` 之后要求特定格式的最终答案。注意：不要让 CoT 的推理过程本身也输出成结构化格式，那会打乱推理的自然流。

**CoT + Few-shot** → 这是 CoT 的"强化版本"。Few-shot 示例中的推理步骤告诉模型你期望的推理粒度和风格，而不只是期望的答案。当 zero-shot CoT 推理步骤过粗或风格不稳定时，用 few-shot CoT 校准。

**CoT vs Prompt Chaining** → 两者都把复杂问题拆分，区别在于：CoT 在单次 LLM 调用里完成拆分和推理；Prompt Chaining 把拆分出的子任务分成多次调用，每次调用的输出作为下次的输入。CoT 更快更简单；Prompt Chaining 每步可以验证、重试、加工具，适合步骤间有分支或需要外部操作的任务。

## 模型差异

| 模型 | 表现描述 | 注意事项 |
|------|---------|---------|
| Claude 4 | 原生支持 extended thinking，推理链质量高且结构清晰；自发使用 `<thinking>` 标签 | extended thinking 模式下推理 token 额外计费；普通模式加 "Let's think step by step" 同样有效 |
| GPT-4 / o1 / o3 | o1/o3 系列内置推理模式，不需要在 prompt 中显式要求 CoT；GPT-4 标准版需要显式触发 | o1/o3 的推理链是内部的（不可见），调试困难；需要可见推理链时用 GPT-4 + 显式 CoT |
| Gemini 2 | Gemini 2.0 Flash Thinking 和 2.0 Pro 支持原生推理模式；标准模式下 zero-shot CoT 效果稳定 | Flash Thinking 的推理链可见但格式略有不同；长推理链下偶有"推理循环"现象，需要设置步骤上限 |
| 开源模型（LLaMA 3、Qwen 2.5、DeepSeek） | 70B+ 参数模型 few-shot CoT 效果较好；小于 7B 的模型 CoT 收益有限甚至负向 | 开源模型对 "Let's think step by step" 的响应不如闭源稳定；建议用 few-shot CoT 而非 zero-shot；DeepSeek-R1 内置推理能力是例外 |

## 常见踩坑

**推理步骤和最终答案混在一起** — 现象：模型在推理过程中就给出结论，后续步骤又推翻，输出前后矛盾。原因：没有明确分隔推理区和结论区。避免方式：用 `<thinking>...</thinking>` 包裹推理，最后单独要求 "Final answer:"，或在 prompt 中明确说"先完成所有推理步骤，最后才给出答案"。

**Few-shot 示例的推理路径误导模型** — 现象：模型照搬示例的推理框架，即使不适用于新问题。原因：LLM 的 in-context learning 会过度拟合示例的推理风格。避免方式：few-shot 示例要覆盖多种推理路径，不要只提供同一种解题套路；或在示例后加说明"以上是示例格式，请根据实际问题灵活运用"。

**用于简单问题导致过度推理** — 现象：模型对"1+1=?"也写出三步推理，输出冗长且看起来很蠢。原因：CoT 指令被无差别应用。避免方式：在 system prompt 中加条件触发，如"只对需要多步推理的问题展示推理步骤，简单问题直接回答"；或在 agent 中根据任务类型动态选择是否启用 CoT。

**推理步骤本身出错但模型没发现** — 现象：中间某步计算错了，后续步骤基于错误结果继续推理，最终答案错误但看起来逻辑自洽。原因：模型不会自动验证中间步骤。避免方式：对于精度要求高的任务，加上 self-consistency（多次采样取多数）；或在 prompt 中加"完成推理后，检查每一步是否有算术错误"；或引入 code interpreter 执行计算部分。

## 来源

- Wei, J. et al. (2022). *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. NeurIPS 2022. https://arxiv.org/abs/2201.11903
- Kojima, T. et al. (2022). *Large Language Models are Zero-Shot Reasoners*. NeurIPS 2022. https://arxiv.org/abs/2205.11916
- Wang, X. et al. (2022). *Self-Consistency Improves Chain of Thought Reasoning in Language Models*. ICLR 2023. https://arxiv.org/abs/2203.11171
- Zhang, Z. et al. (2022). *Automatic Chain of Thought Prompting in Large Language Models*. ICLR 2023. https://arxiv.org/abs/2210.03493
- Prompt Engineering Guide (DAIR.AI). Chain-of-Thought Prompting. https://www.promptingguide.ai/techniques/cot
- 关联 wiki: [[wiki/prompt-system]] [[wiki/query-loop]]
