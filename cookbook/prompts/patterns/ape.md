---
pattern: ape
category: meta
tags: [automatic-prompt-engineering, optimization, prompt-search, llm-as-optimizer]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: advanced
related_wiki: [wiki/prompt-system, wiki/evaluation-observability]
related_patterns: [meta-prompting, few-shot]
---

# APE（Automatic Prompt Engineer）

> 用 LLM 自动搜索和优化 prompt 指令，找到比人类手写更优的 prompt 表达方式。

## 本质

人写 prompt 靠直觉，靠经验，靠反复试错——但人的搜索空间非常有限。APE 的出发点是：**prompt 工程本质上是一个搜索问题**，可以让 LLM 来搜索。

Zhou et al. (2022) 的核心流程：
1. 给 LLM 看 input-output 示例（告诉它这个任务要做什么）
2. 让 LLM 生成一批候选指令（"这个任务的指令可能是什么"）
3. 在目标模型上执行每条候选指令，评测得分
4. 选得分最高的指令

APE 的一个著名发现：自动搜索到的 zero-shot CoT 触发语"Let's work this out in a step by step way to be sure we have the right answer."在 MultiArith 和 GSM8K 上优于人工写的"Let's think step by step."

**核心洞察**：人类觉得两句话差不多，但模型对措辞极其敏感。自动搜索能发现人类直觉忽略的微小语言差异带来的性能差异。

## 什么时候用

**高频使用的关键 prompt** — system prompt、核心任务指令等每天触发数万次的 prompt，值得花计算资源优化。

**性能瓶颈在指令表达时** — 你已经做了 few-shot、CoT，但效果还是卡住了，怀疑是指令措辞问题，用 APE 系统性搜索更好的表达。

**有明确评测指标时** — APE 需要可量化的分数来比较候选指令。分类准确率、生成 BLEU/ROUGE、代码通过率等都可以用。

**跨模型迁移时** — 为模型 A 优化好的 prompt 换到模型 B 效果下降，用 APE 为模型 B 重新搜索最适合它的指令措辞。

## 什么时候不用

**一次性任务** — 只用一次的 prompt，优化收益覆盖不了搜索成本。

**没有评测集** — APE 必须有一个可以打分的评测集。没有评测标准，无法比较候选指令的优劣。

**任务定义本身不清晰时** — 如果你自己都不确定任务目标是什么，生成候选指令也是无的放矢。先明确任务，再用 APE 优化指令。

**需要特定风格或约束的指令** — APE 搜索出来的指令是针对性能优化的，可能不满足品牌语调、合规要求等约束。需要在候选生成阶段加入约束条件。

## 模板

### 基础 APE 流程

```python
# Step 1: 用示例生成候选指令
INSTRUCTION_GENERATION_PROMPT = """
以下是一个任务的输入-输出示例：

{examples}

请生成 10 条不同的指令，每条指令都能描述这个任务的目标。
指令应该简洁、清晰、可操作。
每条指令单独一行，以数字编号开头。
"""

# Step 2: 评测每条候选指令
def evaluate_instruction(instruction: str, eval_set: list, model) -> float:
    scores = []
    for item in eval_set:
        prompt = f"{instruction}\n\n输入：{item['input']}"
        output = model.generate(prompt, temperature=0.0)
        scores.append(compute_score(output, item['expected']))
    return sum(scores) / len(scores)

# Step 3: 选最优指令
def run_ape(examples: list, eval_set: list, generator_model, target_model) -> str:
    # 生成候选
    gen_prompt = INSTRUCTION_GENERATION_PROMPT.format(
        examples=format_examples(examples)
    )
    candidates_text = generator_model.generate(gen_prompt)
    candidates = parse_numbered_list(candidates_text)
    
    # 评测
    scored = [
        (instr, evaluate_instruction(instr, eval_set, target_model))
        for instr in candidates
    ]
    
    # 排序，返回最优
    scored.sort(key=lambda x: x[1], reverse=True)
    return scored[0][0]
```

---

### 迭代搜索版（ResAP / 变体）

```python
def iterative_ape(seed_instruction: str, eval_set: list, model, n_rounds: int = 3) -> str:
    """
    迭代优化：每轮用当前最优指令生成改进变体。
    类似局部搜索，比一次性生成更稳定。
    """
    current_best = seed_instruction
    current_score = evaluate_instruction(current_best, eval_set, model)
    
    for round_i in range(n_rounds):
        # 基于当前最优生成变体
        improve_prompt = f"""
当前指令：{current_best}
当前得分：{current_score:.3f}

请生成 5 个改进版本，尝试不同的措辞、角度或结构，以提升任务性能。
"""
        variants_text = model.generate(improve_prompt)
        variants = parse_numbered_list(variants_text)
        
        # 评测变体
        for variant in variants:
            score = evaluate_instruction(variant, eval_set, model)
            if score > current_score:
                current_best = variant
                current_score = score
    
    return current_best
```

---

### 轻量版：prompt rewriting（无需评测集）

```text
我现在有一条 prompt 指令，效果一般：
{{current_instruction}}

请生成 5 个改进版本，从以下维度改写：
1. 更具体、更有操作性
2. 换一个角度描述任务目标
3. 加入约束条件使输出更可控
4. 更简洁（去掉冗余）
5. 换用更有权威感/专业感的措辞

每个版本独立输出，标注改写维度。
```

## 组合与选择

**APE + Few-shot** — APE 优化指令部分（system prompt / task instruction），few-shot 示例部分保持不变或单独优化。两者互补：APE 管"怎么说任务"，few-shot 管"给什么示范"。

**APE + Self-Consistency** — 用 Self-Consistency 评测候选指令的得分（多次采样取多数），比单次评测更稳定，减少随机噪声对指令排序的影响。评测成本上升，但筛选精度更高。

**APE vs 手动 Prompt 工程** — 不是替代关系。手动调 prompt 快速、成本低、有人类直觉加持；APE 系统性、覆盖更大搜索空间。实践中：先手动调到一个不错的基线，再用 APE 在此基础上精调。

**APE vs Prefix Tuning / Prompt Tuning** — APE 搜索的是**自然语言指令**（可读可解释）；Prefix/Prompt Tuning 优化的是**连续向量**（不可读，但更灵活）。如果不能微调模型参数，用 APE；如果可以微调且追求极致性能，考虑 Prompt Tuning。

**相关工作**：
- **OPRO**（Google, 2023）：用 LLM 作为优化器，在 prompt 里追加优化历史，让 LLM "Take a deep breath" 提升数学题性能
- **AutoPrompt**：基于梯度引导搜索 prompt token（需要模型梯度，适合白盒场景）
- **Prompt-OIRL**：用离线逆强化学习生成 query-dependent 的动态 prompt

## 模型差异

| 模型 | 表现 | 注意事项 |
|------|------|---------|
| Claude 4 | 候选指令生成质量高，多样性好；作为生成器和目标模型都有效 | 生成的候选倾向于冗长详细，建议在生成 prompt 里加"请简洁"约束 |
| GPT-4 | 经典 APE 论文使用 GPT 系列验证；候选指令质量稳定，措辞自然 | o1/o3 更适合作为评测模型而非生成候选，因为其推理模式与指令搜索目标不完全契合 |
| Gemini 2 | Flash 模型作为生成器性价比高（速度快成本低）；Pro 作为评测模型更准确 | Flash 生成的候选多样性略低，可适当提高 temperature |
| 开源模型 | 70B+ 作为候选生成器可用；作为目标模型评测时注意温度设置一致性 | 小模型生成的候选质量差，筛选价值有限 |

## 常见踩坑

**评测集太小导致噪声大** — 用 5 个样本评测候选指令，结果高度不稳定。建议评测集至少 50-100 个样本，确保排序可靠。

**评测集和训练集分布不同** — APE 优化的指令在评测集上最优，但在真实场景下可能过拟合评测分布。使用代表真实场景的评测集，避免"刷榜"。

**候选指令语义重复** — 生成 10 条候选，8 条意思差不多，搜索空间实际上很小。解决：在生成 prompt 里强调多样性，或用聚类去重后再评测。

**用同一个模型生成候选并评测** — 存在"自我强化"偏差：模型生成自己更擅长执行的指令。理想情况下生成器和目标模型可以不同（用一个强模型生成候选，为目标模型优化）。

**忽略指令的泛化性** — APE 找到的最优指令针对评测集，但可能对边缘案例失效。选定最优指令后，仍需在更广泛的测试集上验证泛化性。

## 来源

- Zhou, Y. et al. (2022). *Large Language Models Are Human-Level Prompt Engineers*. ICLR 2023. https://arxiv.org/abs/2211.01910
- Yang, C. et al. (2023). *Large Language Models as Optimizers (OPRO)*. https://arxiv.org/abs/2309.03409
- Shin, T. et al. (2020). *AutoPrompt: Eliciting Knowledge from Language Models with Automatically Generated Prompts*. EMNLP 2020. https://arxiv.org/abs/2010.15980
- Kojima, T. et al. (2022). *Large Language Models are Zero-Shot Reasoners*. NeurIPS 2022. https://arxiv.org/abs/2205.11916
- Prompt Engineering Guide (DAIR.AI). Automatic Prompt Engineer. https://www.promptingguide.ai/techniques/ape
- 关联 wiki: [[wiki/prompt-system]] [[wiki/evaluation-observability]]
