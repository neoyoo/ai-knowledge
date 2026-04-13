---
pattern: active-prompt
category: meta
tags: [example-selection, uncertainty, adaptive, dynamic-examples]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: advanced
related_wiki: [wiki/prompt-system, wiki/evaluation-observability]
related_patterns: [few-shot, chain-of-thought, self-consistency]
---

# Active-Prompt

> 根据模型的不确定性主动选择最有价值的示例加入 prompt，而非随机或人工选择。

## 本质

Few-shot CoT 的质量很大程度上取决于示例选哪些。人工选示例靠直觉，随机选示例靠运气——两种方式都没有利用"这些示例对这个模型到底有没有用"的信号。

Active-Prompt 的核心思路来自主动学习（active learning）：**找出模型最不确定的问题，把这些当示例**。理由：如果模型对某类问题拿不准，给它这类问题的示范，学习价值最高；对已经会的问题做示范，收益接近零。

具体流程（Diao et al., 2023）：
1. 用模型对一批训练问题各生成 k 个答案（不同采样）
2. 计算每个问题的不确定度（k 个答案的分歧程度）
3. 选出不确定度最高的问题，交人工标注 CoT 推理链
4. 把这些高价值标注示例作为 few-shot prompt

结果是：花同样的标注成本，得到比随机选示例更好的 prompt。

## 什么时候用

**有标注预算时优化示例质量** — 你能标注的示例数量有限（比如只能标 8 个），但训练集有 1000 个问题。Active-Prompt 告诉你标哪 8 个最值钱。

**跨任务类型的通用 few-shot** — 任务类型多样、问题难度不均时，随机示例可能覆盖不到难点。Active-Prompt 确保示例覆盖模型的薄弱区域。

**评测套件/基准任务** — 在数学推理、常识问答等有明确评测集的任务上，Active-Prompt 可以系统性地优化示例选择，比人工选示例更可靠。

**迭代优化 prompt** — 当前 few-shot prompt 效果一般，想知道换哪些示例能提升，用不确定度分析找方向。

## 什么时候不用

**没有训练集或标注资源** — Active-Prompt 需要有一批候选问题和人工标注能力。纯零-shot 场景无法应用。

**单次任务、没有复用价值** — 如果这个 prompt 只用一次，优化示例选择的成本不值得。适合高频使用、需要持续优化的 prompt。

**任务类型极度统一** — 问题同质化程度很高时，随机选和主动选差别不大，Active-Prompt 的收益有限。

**延迟要求极高的场景** — 不确定度计算需要多次采样（k 次推理），这是离线优化步骤，不影响线上推理延迟。但如果连离线准备时间都不允许，也无法用。

## 模板

### 不确定度计算（离线步骤）

```python
import collections

def compute_uncertainty(model, question: str, k: int = 10) -> float:
    """
    生成 k 个答案，用答案分歧度衡量不确定度。
    分歧度越高 = 模型越不确定 = 示例价值越高。
    """
    answers = []
    for _ in range(k):
        response = model.generate(question, temperature=0.7)
        answers.append(extract_answer(response))  # 提取最终答案
    
    # 计算熵：答案越分散，不确定度越高
    counter = collections.Counter(answers)
    total = len(answers)
    entropy = -sum((c/total) * (c/total).bit_length() for c in counter.values())
    
    # 简化版：用"最多数答案的占比"衡量（占比越低 = 越不确定）
    most_common_ratio = counter.most_common(1)[0][1] / total
    return 1 - most_common_ratio  # 越高越不确定


def select_examples_by_uncertainty(questions: list, model, n_select: int = 8) -> list:
    """选出不确定度最高的 n 个问题用于标注"""
    scored = [(q, compute_uncertainty(model, q)) for q in questions]
    scored.sort(key=lambda x: x[1], reverse=True)
    return [q for q, _ in scored[:n_select]]
```

---

### 标注提示（交给人工标注者）

```text
以下问题是模型最不确定的，请提供详细的推理过程和最终答案。

推理格式：
Q: {{question}}
A: [逐步推理过程]
   因此，答案是：[最终答案]

---
待标注问题：
{{uncertain_question}}
```

---

### 最终 few-shot prompt（线上使用）

```text
以下是几个示例，展示如何解答这类问题：

{{annotated_example_1}}

---

{{annotated_example_2}}

---

{{annotated_example_3}}

---

现在请回答：
Q: {{new_question}}
A:
```

---

### 轻量版：无人工标注，用 LLM 自动标注不确定示例

```python
def auto_annotate_uncertain(model, uncertain_questions: list) -> list:
    """
    无法人工标注时，用更强的模型对不确定问题生成 CoT 示例。
    注意：自动标注质量低于人工，但比随机选示例仍有提升。
    """
    annotated = []
    for q in uncertain_questions:
        cot_response = model.generate(
            f"请详细展示解题步骤：{q}",
            system="你是一个准确、细致的推理助手，每步都要写清楚",
            temperature=0.0  # 低温度保证标注稳定性
        )
        annotated.append({"question": q, "cot": cot_response})
    return annotated
```

## 组合与选择

**Active-Prompt + CoT** — Active-Prompt 决定**选哪些示例**，CoT 决定**示例怎么写**。两者是正交的：先用 Active-Prompt 选出高价值问题，再为这些问题写 CoT 推理链作为示例。

**Active-Prompt + Self-Consistency** — Self-Consistency 用多次采样提升单个问题的答案质量；Active-Prompt 用多次采样识别哪些问题需要更好的示例。可以复用同一批采样结果：既用来计算不确定度（选示例），也用来取多数答案（提升精度）。

**Active-Prompt vs 随机 Few-shot** — 随机选示例是 baseline，Active-Prompt 是系统化优化。当标注预算有限时，Active-Prompt 能把有限资源用在最有价值的地方。

**Active-Prompt vs Auto-CoT** — Auto-CoT 用聚类选示例（覆盖多样性），Active-Prompt 用不确定度选示例（聚焦难点）。前者保证多样性，后者聚焦薄弱点。可以结合：先聚类保证多样性，再在每类中选不确定度最高的。

## 模型差异

| 模型 | 表现 | 注意事项 |
|------|------|---------|
| Claude 4 | 不确定度计算稳定，k=8-10 次采样即可得到可靠的分歧信号 | extended thinking 模式下 temperature 控制不同，不确定度采样需要用标准模式 |
| GPT-4 | 不确定度估计准确，o1/o3 因为内置推理机制答案更一致，分歧度较低 | o1/o3 系列用于不确定度计算效果较差（答案太一致），建议用 GPT-4 标准版 |
| Gemini 2 | Flash 模型不确定度计算成本低，适合大批量候选问题筛选 | Flash 的分歧信号比 Pro 噪声更大，建议 k≥15 |
| 开源模型 | 70B+ 模型不确定度信号有效；小模型（<13B）分歧度普遍偏高，难以区分真正不确定和随机波动 | 建议用小模型生成候选、大模型验证不确定度 |

## 常见踩坑

**k 太小导致不确定度估计不稳定** — k=3 时随机性很大，不确定度排序不可靠。建议 k≥8，平衡采样成本和估计稳定性。

**把"罕见问题"当"不确定问题"** — 如果训练集分布不均，某类问题出现频率低，模型可能对它们不确定，但标注这些对提升整体性能帮助有限。建议结合数据分布分析不确定度来源。

**不确定度计算和最终评测用同一个模型** — 如果用 GPT-4 计算不确定度，也用 GPT-4 做最终推理，这是合理的；但如果目标模型是另一个（如开源模型），应该用目标模型计算不确定度。

**忽略标注质量** — Active-Prompt 只是选出了"最值得标注的问题"，标注本身的质量还是由人（或自动标注模型）决定。劣质标注加上主动选择，效果可能比随机选示例加高质量标注更差。

**把 Active-Prompt 用于线上实时场景** — Active-Prompt 的不确定度计算是离线步骤，用于优化 few-shot 示例集。线上推理时直接用优化好的示例，不需要每次都重新计算。

## 来源

- Diao, S. et al. (2023). *Active Prompting with Chain-of-Thought for Large Language Models*. ACL 2024. https://arxiv.org/abs/2302.12246
- Wei, J. et al. (2022). *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. NeurIPS 2022. https://arxiv.org/abs/2201.11903
- Wang, X. et al. (2022). *Self-Consistency Improves Chain of Thought Reasoning in Language Models*. ICLR 2023. https://arxiv.org/abs/2203.11171
- Prompt Engineering Guide (DAIR.AI). Active-Prompt. https://www.promptingguide.ai/techniques/activeprompt
- 关联 wiki: [[wiki/prompt-system]] [[wiki/evaluation-observability]]
