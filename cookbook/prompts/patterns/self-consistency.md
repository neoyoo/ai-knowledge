---
pattern: self-consistency
category: reasoning
tags: [sampling, majority-vote, reliability, reasoning-verification]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_wiki: [wiki/prompt-system, wiki/evaluation-observability]
related_patterns: [chain-of-thought, react]
---

# Self-Consistency

> 对同一问题用 CoT 生成多条推理路径，取多数一致的答案，用冗余换可靠性。

## 本质

想象你要做一道难题，做了一遍不确定答案对不对，于是换个思路再做一遍，再换一遍。如果三次里有两次都得出同一个答案，你会更有信心这个答案是对的。Self-Consistency 对 LLM 做的正是这件事——"三个臭皮匠顶一个诸葛亮"。

单次 CoT 推理是有概率出错的：推理链走错了一步，后续都会跑偏，而且模型自己往往意识不到。Self-Consistency 的解法是：用 temperature > 0 让模型产生多样化的推理路径，然后取多数答案作为最终输出。关键在于：不同的推理路径出现**相同错误**的概率远低于单条路径出错的概率，所以多数投票的结果更可靠。

这个方法来自 Wang et al. (2022)，初衷是替代 CoT 中的贪心解码（greedy decoding）。在数学和常识推理任务上，Self-Consistency 能在 CoT 基础上再提升 10-20 个百分点的准确率。

## 什么时候用

**数学与定量推理** — 多步计算中任何一步出错都会导致最终答案错误，单次推理风险高。多路采样能捕捉不同的计算路径，过滤偶发性算术错误。

**逻辑推断与常识推理** — 问题有明确唯一的正确答案，推理路径多种多样但殊途同归。多数投票能过滤走偏的推理链。

**高准确率要求的生产场景** — 当一次错误的代价很高（如代码生成中的关键逻辑判断、合规检查、医疗问答），用 N 倍成本换取更高的可靠性是合理的取舍。

**Agent 关键决策节点** — Agent 在一个高风险决策点（如选择不可逆操作、生成用于外部调用的参数）时，对该节点多路推理后投票，而不是整条链路都做 N 次。

## 什么时候不用

**开放式生成任务** — 写故事、写文案、头脑风暴没有"唯一正确答案"，投票机制无法工作。这类任务用 Best-of-N + 质量评分器更合适。

**延迟敏感的实时应用** — N 次并行调用虽然可以并发，但仍有额外 API 开销和尾延迟。流式对话、交互式产品优先考虑单次 CoT。

**成本敏感的高频场景** — 成本直接乘以 N。如果每天有百万次调用，5x 采样意味着 5x 成本，需要仔细测算 ROI。

**答案不可比较的场景** — 如果多次输出的格式、粒度不一致，无法做多数投票（例如生成代码片段而非离散答案）。这种情况需要先设计输出标准化步骤。

## 模板

### 基础版：同一 Prompt 运行 N 次 + 投票

Self-Consistency 不改变 prompt 本身，关键在于**调用方式**：

```python
import anthropic
from collections import Counter

client = anthropic.Anthropic()

def self_consistency(prompt: str, n: int = 5, temperature: float = 0.7) -> str:
    """
    对同一问题采样 n 条推理路径，取多数答案。
    """
    answers = []

    for _ in range(n):
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1024,
            temperature=temperature,
            messages=[{"role": "user", "content": prompt}]
        )
        raw = response.content[0].text

        # 从推理链末尾提取最终答案
        # 约定：prompt 要求模型以 "The answer is X" 结尾
        answer = extract_final_answer(raw)
        answers.append(answer)

    # 多数投票
    vote_counts = Counter(answers)
    majority_answer, count = vote_counts.most_common(1)[0]
    confidence = count / n

    return majority_answer, confidence, answers


def extract_final_answer(text: str) -> str:
    """从推理文本中提取标准化答案。根据任务格式调整。"""
    import re
    match = re.search(r"[Tt]he answer is\s+(.+?)[\.\n]", text)
    if match:
        return match.group(1).strip()
    # 兜底：取最后一行
    return text.strip().split("\n")[-1].strip()
```

配套的 prompt 模板（要求模型以统一格式给出最终答案，便于提取和投票）：

```text
{{problem_statement}}

Let's think step by step. At the end, state your final answer as: "The answer is [answer]."
```

示例调用：

```python
prompt = """
When I was 6, my sister was half my age. Now I'm 70, how old is my sister?

Let's think step by step. At the end, state your final answer as: "The answer is [answer]."
"""

answer, confidence, all_answers = self_consistency(prompt, n=5)
print(f"Final answer: {answer} (confidence: {confidence:.0%})")
print(f"All votes: {all_answers}")
```

典型输出（展示三条不同推理路径殊途同归）：

```text
# 路径 1
When I was 6, my sister was half my age, so she was 3.
The age gap between us is 6 - 3 = 3 years.
Now I'm 70, so my sister is 70 - 3 = 67.
The answer is 67.

# 路径 2
At age 6, sister was 3 (half of 6). Age difference = 3 years.
Sister is always 3 years younger. 70 - 3 = 67.
The answer is 67.

# 路径 3 (走偏的路径)
My sister was half my age when I was 6, so she was 3.
Now I'm 70, so she is 70/2 = 35.
The answer is 35.

# 投票结果：67 获得 2 票，35 获得 1 票
# 多数答案：67 ✓
```

---

### Agent 集成版：关键决策节点多路验证

不需要对整条 agent 链路做 N 次，只在**高风险决策节点**做 Self-Consistency：

```python
async def agent_critical_decision(
    context: str,
    decision_prompt: str,
    n: int = 3
) -> str:
    """
    在 agent 的关键决策点（如选择工具、生成 SQL、判断条件）
    对该节点多路采样后投票，其余节点正常单次调用。
    """
    answer, confidence, votes = self_consistency(
        prompt=f"{context}\n\n{decision_prompt}",
        n=n,
        temperature=0.6
    )

    if confidence < 0.6:
        # 置信度不足时，记录日志或交由人工审核
        log_low_confidence_decision(decision_prompt, votes, confidence)

    return answer
```

在 agent system prompt 中标记需要多路验证的决策类型：

```text
你是一个数据分析 agent。你的任务是回答用户的数据问题。

对于以下类型的判断，你必须在 <critical_decision> 标签中给出答案，
系统会对这类判断进行多路验证：
- 涉及数值计算的最终结论
- SQL 查询条件的逻辑判断
- 异常数据的分类判断

普通步骤（数据读取、格式转换等）正常执行，无需多路验证。
```

---

### 变体：Universal Self-Consistency (USC)

当答案是开放式文本（无法直接投票）时，用 LLM 自身做评判：

```python
def universal_self_consistency(prompt: str, n: int = 5) -> str:
    """
    生成 N 条候选回答，再用 LLM 找出最一致的那个。
    适用于无法直接多数投票的开放式问题。
    来自 Chen et al. (2023)。
    """
    candidates = []
    for _ in range(n):
        response = call_llm(prompt, temperature=0.8)
        candidates.append(response)

    # 用 LLM 做最终裁决
    judge_prompt = f"""
以下是对同一问题的 {n} 条不同回答：

{chr(10).join(f'{i+1}. {c}' for i, c in enumerate(candidates))}

哪条回答最准确、最符合多数回答的共识？
请直接输出最佳回答的完整内容，不要编号或额外说明。
"""
    return call_llm(judge_prompt, temperature=0.0)
```

## 关键参数

**采样次数 N** — 通常 5-10 次可以覆盖大多数场景。N=3 是最低有效门槛（2:1 才能多数胜出）；N=5 是常用平衡点；N=10+ 适用于最高精度要求但成本很高。经验上，准确率提升在 N=5 附近开始边际递减，继续增加 N 的收益有限。

**Temperature 设置** — 需要 > 0 才能产生多样化路径。太低（如 0.1）几乎每次都得到相同推理，等于浪费 N 倍成本；太高（如 1.5）推理质量下降，噪音太多。推荐范围：0.5–0.8。数学推理任务可以用 0.5-0.7；常识推理可以稍高 0.7-0.9。

**投票策略** — 最简单的是直接多数投票（plurality voting）。对于有置信度信息的场景，可以加权投票——让模型同时输出答案和置信度，置信度更高的路径权重更大。但实践中简单多数投票已经效果很好，加权投票收益不明显。

**答案标准化** — 投票前要做标准化，否则同一答案的不同写法会被当成不同选项（"67 岁" vs "67" vs "sixty-seven"）。正则提取 + 类型归一化是必须的预处理步骤。

## 组合与选择

**Self-Consistency + CoT** — 这是最标准的组合，Self-Consistency 本身就是建立在 CoT 基础上的。每条推理路径本身就是一次 CoT，多路投票在 CoT 基础上再加一层鲁棒性。不需要修改 prompt，只改调用方式。

**Self-Consistency vs ReAct** — 两者解决不同问题。ReAct 通过工具调用获取外部信息来提升推理准确率；Self-Consistency 通过冗余推理过滤随机错误。两者可以组合：对 ReAct agent 的关键推理节点（而非工具调用节点）做 Self-Consistency。

**Self-Consistency vs Prompt Chaining** — Prompt Chaining 把一个大问题拆成多步顺序执行，每步结果流向下一步；Self-Consistency 是同一步骤并行执行 N 次然后投票。如果问题可以分解，先用 Prompt Chaining 降低单步复杂度，再在每步内部视情况用 Self-Consistency。

**Self-Consistency vs Best-of-N** — Best-of-N 生成 N 个候选，用外部评分器（验证器、奖励模型）选最好的一个；Self-Consistency 用多数投票，无需外部评分器。Self-Consistency 更简单，适合有明确正确答案的任务；Best-of-N 适合开放式生成，但需要额外建设评分器。

## 模型差异

| 模型 | 表现描述 | 注意事项 |
|------|---------|---------|
| Claude 4 | 推理链质量高，不同采样路径的多样性较好；extended thinking 模式下每条路径的推理更深入 | extended thinking 模式成本更高，Self-Consistency 叠加后成本可观；普通模式已足够 |
| GPT-4o | 多次采样的路径多样性良好，多数投票效果稳定；o1/o3 内置推理已有类似效果 | o1/o3 系列内置了类似 Self-Consistency 的机制，外部再做 N 次采样收益有限；GPT-4o 标准版用 Self-Consistency 提升明显 |
| Gemini 2 | Flash 模型成本低，适合做更多采样次数（N=10+）；Pro 模型单次质量高，N=5 足够 | Flash Thinking 模式本身有多路推理，外部 Self-Consistency 可能重复 |
| 开源模型（LLaMA 3、Qwen 2.5、DeepSeek） | 70B+ 模型效果明显，单次推理错误率较高时 Self-Consistency 提升幅度更大 | 小模型（<13B）单次推理质量太低，多数路径都错时投票也救不了；Self-Consistency 对基础推理能力有最低要求 |

## 常见踩坑

**所有路径都走向同一个错误** — 现象：N 次采样全部得出同一个错误答案，投票结果仍然错。原因：temperature 太低导致多样性不足，或问题存在系统性陷阱（所有路径都会踩的常识错误）。避免方式：用更高的 temperature（0.7+）确保路径多样性；对于已知有系统性陷阱的问题类型，改用 few-shot CoT 提供正确推理示范，再叠加 Self-Consistency。

**Temperature 太低，采样结果几乎相同** — 现象：5 次采样返回几乎一致的文本，等于只做了一次推理。原因：temperature 设置过低（如 0.1 或 0.0）。避免方式：temperature 至少设置到 0.5，推荐 0.7。检查方式：打印所有采样结果，确认推理路径有实质差异。

**答案提取不稳定导致投票失效** — 现象：明明 3 条路径都得出正确答案，但因为答案格式不一致（"答案是 67"、"67 岁"、"the answer: 67"），被分散成三个不同选项，都只得 1 票。避免方式：在 prompt 中严格规定输出格式（"以 'The answer is X' 结尾"）；写健壮的答案提取正则；在投票前做标准化（转小写、去标点、数字统一格式）。

**成本失控** — 现象：开发时 N=10 效果很好，上线后账单暴增。原因：没有在全流程中对 Self-Consistency 节点的成本做预算和监控。避免方式：只在真正需要高准确率的节点用 Self-Consistency，其余节点单次调用；设置 N 的上限；记录每个 Self-Consistency 调用的 token 消耗；考虑用更便宜的模型做多次采样。

**并发调用未做超时控制** — 现象：N 次并行调用中某一次超时，整个 Self-Consistency 挂起等待。避免方式：对每次子调用设置独立 timeout；设计为"N 次中先完成的 M 次（M < N）就开始投票"的 early-exit 策略，既降低延迟又保证基本可用性。

## 来源

- Wang, X. et al. (2022). *Self-Consistency Improves Chain of Thought Reasoning in Language Models*. ICLR 2023. https://arxiv.org/abs/2203.11171
- Chen, X. et al. (2023). *Universal Self-Consistency for Large Language Model Generation*. https://arxiv.org/abs/2311.17311
- Wei, J. et al. (2022). *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. NeurIPS 2022. https://arxiv.org/abs/2201.11903
- Prompt Engineering Guide (DAIR.AI). Self-Consistency. https://www.promptingguide.ai/techniques/consistency
- 关联 wiki: [[wiki/prompt-system]] [[wiki/evaluation-observability]]
- 关联 pattern: [[cookbook/prompts/patterns/chain-of-thought]] [[cookbook/prompts/patterns/react]]
