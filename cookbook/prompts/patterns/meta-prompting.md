---
pattern: meta-prompting
category: meta
tags: [prompt-generation, prompt-optimization, self-improvement, automation]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: advanced
related_wiki: [wiki/prompt-system]
related_patterns: [ape, system-prompt-design, chain-of-thought]
---

# Meta Prompting

> 用 LLM 来生成、改进或优化 prompt 本身，让模型成为 prompt 工程师。

## 本质

Meta Prompting 的核心思路是：不自己写 prompt，让模型帮你写。"用 AI 优化 AI 的输入"。

它有两种含义，经常混用，要分清：

**含义一（Zhang et al. 2024，偏学术）**：关注 prompt 的结构和语法模式，而非具体内容。给模型一个抽象的结构模板（类型、格式、逻辑框架），让模型在结构约束下泛化到不同问题，不依赖具体 few-shot 示例。本质是"描述问题的形状，而非给出示例"。

**含义二（工程实践，更常见）**：把 prompt 本身当作优化目标，让 LLM 生成新 prompt、改写现有 prompt、或在自动循环中迭代优化 prompt。这是 APE（Automatic Prompt Engineer）等技术的基础。

两种含义都指向同一个根本转变：**把 prompt 从"人工制品"变成"模型输出"**，解放人工成本，同时让优化可以程序化。

## 什么时候用

**批量生成 prompt 变体** — 需要测试 10 种不同表述方式时，手工写太慢。让模型生成变体，再批量评估，效率提升 10 倍以上。

**优化现有 prompt 性能** — 你有一个能用但不够好的 prompt，想提升它的准确率或输出质量。描述问题让模型改写，通常比自己反复调整更快找到改进方向。

**探索未知领域的 prompt 写法** — 进入新任务域（比如法律文本分析、代码审查）时，不知道从哪里入手。让模型先生成一批候选 prompt，你来筛选和调整。

**自动化 prompt 工程管道** — 在评估框架中，需要对 prompt 进行程序化迭代改进。Meta Prompting 配合评估指标，可以构建自动优化循环（类 DSPy 方案）。

**结构化提问的泛化** — 面对同一类型的不同问题（如各种数学题），与其构造 n 条 few-shot 示例，不如给一个抽象结构模板，让模型在结构约束下泛化，更节省 token。

## 什么时候不用

**简单明确的任务** — 如果任务一句话就能说清楚，手写 prompt 更快、更可控。让模型"再生成一个 prompt"只是多绕了一圈。

**需要精确控制 prompt 内容** — 模型生成的 prompt 不可预测，如果你对措辞、格式、信息顺序有严格要求（合规场景、品牌语言），Meta Prompting 会让你失去控制权。

**对 prompt 可解释性有要求** — 如果你需要向用户或审查者解释"为什么这个 prompt 这样写"，模型自动生成的 prompt 很难给出清晰理由。人工设计的 prompt 有明确的设计意图。

**没有评估指标时** — Meta Prompting 生成了更好的 prompt，但"更好"如何衡量？没有评估标准就无法判断优化方向，优化循环会变成随机漫步。这是最常见的失败原因。

**延迟敏感的实时路径** — 用 LLM 生成 prompt 是一次额外的模型调用，增加了整体延迟。在 p99 延迟敏感的生产系统里，这层开销通常不可接受。Meta Prompting 更适合离线优化而非在线推理。

## 模板

### 基础版

**改写/优化现有 prompt** — 给出原始 prompt 和改进目标，让模型生成更好的版本：

```text
你是一个 prompt 工程师。请改进以下 prompt，使其在完成 [任务目标] 时更加清晰、有效。

原始 prompt：
{{original_prompt}}

改进要求：
- 输出格式应该是 [格式要求]
- 需要处理的边界情况：[边界情况列表]
- 不需要改变：[不变的约束]

请给出改进后的 prompt，并简要说明每处改动的原因。
```

示例：

```text
你是一个 prompt 工程师。请改进以下 prompt，使其在提取合同关键条款时更加准确。

原始 prompt：
"分析这份合同，告诉我重要的内容。"

改进要求：
- 输出格式应该是结构化 JSON，包含 parties/dates/obligations/termination 字段
- 需要处理的边界情况：合同条款缺失时需要标注 null 而非省略
- 不需要改变：输入语言（合同可能是中英文混合）

请给出改进后的 prompt，并简要说明每处改动的原因。
```

---

**生成结构化模板（Zhang et al. 风格）** — 给出问题类型的抽象结构，让模型填充或泛化：

```text
以下是一类问题的结构模式：

问题类型：{{problem_type}}
输入格式：{{input_structure}}
期望输出格式：{{output_structure}}
推理路径：{{reasoning_pattern}}

现在请用这个结构框架解决以下具体问题：
{{actual_problem}}
```

---

**生成 prompt 变体** — 用于批量生成候选 prompt 进行 A/B 测试：

```text
请为以下任务生成 5 种不同风格的 prompt，覆盖不同的表述策略：

任务描述：{{task_description}}
目标模型：{{target_model}}
评估指标：{{success_criteria}}

对每种变体，注明：
1. 变体 prompt 本身
2. 核心策略（如 zero-shot / few-shot / role-play / constraint-first 等）
3. 适用场景假设
```

---

### Agent 集成版

在自动 prompt 优化循环中使用 Meta Prompting，构建类 DSPy 的本地迭代方案：

```text
你是一个 prompt 优化器。你的任务是改进一个用于 [任务描述] 的 prompt。

当前 prompt：
{{current_prompt}}

在最近的测试中，该 prompt 表现如下：
- 成功案例：{{success_examples}}
- 失败案例：{{failure_examples}}
- 当前指标：准确率 {{accuracy}}，失败模式：{{failure_patterns}}

请分析失败原因，并生成一个改进版本。改进时遵守以下约束：
- prompt 长度不超过 {{max_tokens}} tokens
- 必须保持与当前格式兼容（输出结构不变）
- 不引入 few-shot 示例（系统有 token 限制）

输出格式：
<analysis>失败原因分析</analysis>
<improved_prompt>改进后的完整 prompt</improved_prompt>
<changes>改动说明（逐条列出）</changes>
```

在 agent 代码中，把这个优化调用封装成一个工具，在评估指标下降时自动触发：

```python
# 伪代码示意
def optimize_prompt(current_prompt, eval_results):
    meta_prompt = build_meta_prompt(current_prompt, eval_results)
    improved = llm.call(meta_prompt)
    new_prompt = parse_improved_prompt(improved)
    # 评估新 prompt，如果更好则替换
    if evaluate(new_prompt) > evaluate(current_prompt):
        return new_prompt
    return current_prompt
```

## 组合与选择

**Meta Prompting + 评估框架** → 这是 Meta Prompting 发挥最大价值的场景。优化方向必须有指标驱动，否则是盲目的。先建立评估集和评估指标，再运行优化循环。这是 DSPy、TextGrad 等自动 prompt 优化框架的底层逻辑。

**Meta Prompting + CoT** → 在 Meta Prompting 生成新 prompt 的步骤本身加上 CoT，让模型解释为什么这样改。这样生成的 prompt 带有"设计意图"说明，便于人工审查和二次调整，不是黑箱。

**Meta Prompting + Few-Shot** → Meta Prompting 可以用来自动生成 few-shot 示例，而不是只改写 prompt 文本。给出任务描述和输入输出格式，让模型生成高质量示例，再手工筛选几条使用。比手工构造示例快得多。

**Meta Prompting vs APE（Automatic Prompt Engineer）** → APE 是 Meta Prompting 的完整自动化实现：生成候选 prompt → 评估 → 选最优 → 迭代。Meta Prompting 是 APE 的核心组件（生成步骤），APE 在外层加了评估和选择循环。如果你只需要生成候选、人工筛选，直接用 Meta Prompting；如果需要全自动，走 APE/DSPy。

**Meta Prompting vs 手工 Prompt Engineering** → 不是替代关系。Meta Prompting 擅长生成和变体探索；手工工程擅长精细控制和可解释性。实践中的工作流是：手工确定大方向和约束 → Meta Prompting 生成候选变体 → 手工筛选和精调 → 评估。

## 模型差异

| 模型 | 表现描述 | 注意事项 |
|------|---------|---------|
| Claude 4 | 生成的 prompt 结构清晰，能理解约束并在改进时给出详细理由；对"不改变什么"的约束遵守较好 | 倾向于生成较长的 prompt；如有 token 限制需明确在 meta prompt 中说明 |
| GPT-4 | 生成变体的多样性较高，适合探索阶段；o1/o3 系列对结构模板任务理解深但输出较慢 | GPT-4 生成的 prompt 有时过度依赖特定格式约束，泛化性稍弱 |
| Gemini 2 | Gemini 2.0 Pro 在结构化 prompt 生成上表现稳定；长上下文场景下能参考更多失败案例 | Flash 系列速度快但生成的 prompt 质量和一致性不如 Pro；优化循环中建议用 Pro |
| 开源模型（LLaMA 3、Qwen 2.5） | 70B+ 模型能完成基础的 prompt 改写任务；但对复杂约束的遵守和"分析失败原因"的质量明显弱于闭源 | 不建议用小于 30B 的模型做 Meta Prompting，生成质量不稳定；Qwen 2.5-72B 是开源中表现较好的选项 |

## 常见踩坑

**生成的 prompt 越来越长** — 现象：每轮优化后 prompt 都变长，最终臃肿不堪，性能反而下降。原因：模型倾向于"加内容"而非"替换内容"，认为更多说明等于更好。避免方式：在 meta prompt 中明确设置 token 上限；或加约束"不得在原有基础上添加超过 X 字"；定期做 prompt 瘦身。

**优化方向不可控** — 现象：你想改进 JSON 格式的稳定性，但模型改了推理风格；你想提升准确率，但模型改了语气。原因：meta prompt 中的改进目标不够具体，模型自行决定优化维度。避免方式：在 meta prompt 中用"只改变 X，保持 Y 和 Z 不变"的约束结构；每次优化循环只针对一个维度。

**没有评估指标就开始优化** — 现象：生成了 5 个 prompt 变体，但不知道选哪个，最后凭感觉选了一个。原因：跳过了最重要的前置步骤——定义成功标准。避免方式：在启动任何 Meta Prompting 流程前，先确定评估集（10-50 条代表性输入）和评估指标（准确率、格式合规率、人工评分等），否则优化是盲目的。

**Meta Prompt 本身写得太差** — 现象：让模型优化 prompt，但生成的结果毫无价值，甚至更差。原因：Meta prompt（用于生成 prompt 的那个 prompt）本身质量低，没有提供足够的上下文和约束。本质是：Meta Prompting 只是把问题上移了一层，meta prompt 本身仍然需要好好写。避免方式：在 meta prompt 中提供：任务描述、目标模型、失败案例、不可改变的约束、输出格式要求。

**陷入优化循环但无法收敛** — 现象：循环跑了 10 轮，指标震荡，没有单调上升的趋势。原因：优化目标有冲突（如同时提升准确率和缩短输出），或评估集太小导致过拟合某几条样本。避免方式：确保评估集有足够覆盖度（至少 30 条以上）；每次只优化一个维度；设置早停条件（连续 3 轮无改善则停止）。

## 来源

- Zhang, T. et al. (2024). *Meta-Prompting: Enhancing Language Models with Task-Agnostic Scaffolding*. https://arxiv.org/abs/2311.11482
- Zhou, Y. et al. (2022). *Large Language Models Are Human-Level Prompt Engineers*. ICLR 2023. https://arxiv.org/abs/2211.01910 （APE 原论文）
- Khattab, O. et al. (2023). *DSPy: Compiling Declarative Language Model Calls into Self-Improving Pipelines*. https://arxiv.org/abs/2310.03714
- Prompt Engineering Guide (DAIR.AI). Meta Prompting. https://www.promptingguide.ai/techniques/meta-prompting
- 关联 wiki: [[wiki/prompt-system]]
