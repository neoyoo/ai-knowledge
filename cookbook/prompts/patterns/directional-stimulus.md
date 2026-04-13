---
pattern: directional-stimulus
category: reasoning
tags: [hint, guidance, stimulus, targeted-prompting]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_wiki: [wiki/prompt-system]
related_patterns: [few-shot, chain-of-thought]
---

# Directional Stimulus Prompting (DSP)

> 在 prompt 中加入定向提示（hint/keyword），引导模型朝特定方向推理，而不是让模型自由发挥。

## 本质

想象老师出完题后说"提示：想想勾股定理"——这句话本身不给答案，但把模型的注意力引到正确的方向上。Directional Stimulus Prompting 做的就是这件事：在 prompt 里插入一个"刺激信号"（hint、keyword、关键词），让模型在生成时往这个方向收敛。

原论文（Li et al., 2023）的做法更进一步：用一个小型策略 LM（policy LM）来**自动生成**这个 hint，然后把 hint 注入到主模型的 prompt 里。策略 LM 可以用 RL 微调，专门优化"生成对主模型有引导价值的提示词"这个目标。实践中，手写 hint 也完全有效，不一定需要训练策略模型。

**与 few-shot 的区别**：few-shot 用示例教模型"格式"；DSP 用 hint 引导模型"方向"。前者是"看我怎么做"，后者是"往这里看"。

## 什么时候用

**摘要/总结任务** — 原论文在摘要任务上验证了 DSP 的效果。提供关键词 hint 能让摘要聚焦在重要信息上，而不是模型自由发挥选主题。

**知识问答** — 当问题有明确的答案域（如某个领域、某个时期、某个视角），用 hint 缩小搜索空间，减少模型"跑偏"。

**受控生成** — 需要输出包含特定概念、风格或立场时，hint 比长篇指令更精准。例如：要求生成"悲观但理性"的分析，直接在 prompt 里加 `[hint: 强调风险和不确定性]`。

**Agent 中的工具选择辅助** — 给 agent 提供上下文提示，如 `[context hint: 用户数据已加载，优先用 data_analysis 工具]`，引导工具选择。

## 什么时候不用

**需要模型发散思维时** — hint 会约束输出方向，创意写作、头脑风暴、开放性探索不适合。

**hint 本身不确定时** — 如果你不清楚正确方向，乱加 hint 会把模型往错误方向引，比不加更差。

**简单直接的任务** — 事实查询、格式转换等，加 hint 增加复杂度而不带来收益。

**需要客观中立输出时** — 法律文书、技术文档等，定向刺激可能引入偏向。

## 模板

### 基础版：手动 hint 注入

```text
{{task_description}}

[关键词提示: {{hint_keywords}}]

{{input_content}}
```

示例（摘要任务）：

```text
请对以下新闻文章写一段 150 字的摘要。

[关键词提示: 经济影响、政策变化、企业反应]

{{article_text}}
```

---

### 结构化 hint 版

```text
任务：{{task}}

方向提示：
- 重点关注：{{focus_area}}
- 视角：{{perspective}}
- 关键概念：{{keywords}}

输入：{{input}}
```

---

### Agent 版：动态 hint 注入

```python
# 根据任务类型动态生成 hint
def build_prompt_with_hint(task, input_text, task_type):
    hints = {
        "summarize": "保留关键数字、时间节点和核心结论",
        "analyze": "识别因果关系、找出假设前提",
        "code_review": "关注安全性、性能瓶颈和边界条件",
    }
    hint = hints.get(task_type, "")
    return f"{task}\n\n[方向提示: {hint}]\n\n{input_text}"
```

---

### 进阶版：策略模型生成 hint（论文方案）

```text
# Step 1: 用小型策略模型生成 hint
策略模型 prompt:
"给定以下输入，生成 3 个关键词来引导摘要方向：
输入: {{input}}
关键词（仅输出，逗号分隔）:"

# Step 2: 将 hint 注入主模型 prompt
主模型 prompt:
"请根据以下关键词方向写摘要：[{{generated_keywords}}]
原文: {{input}}"
```

## 组合与选择

**DSP + CoT** → hint 指定推理方向，CoT 展开推理步骤。先用 hint 锁定范围，再用 CoT 在范围内深挖。适合：知识密集型问题，答案域已知但推理过程复杂。

**DSP + Few-shot** → 示例展示推理格式，hint 引导具体方向。两者互补：few-shot 管"怎么想"，DSP 管"想什么"。

**DSP vs System Prompt 约束** — 两者都能约束输出方向。区别：System Prompt 约束是全局的、持久的；DSP hint 是针对单次任务的、局部的。需要全局行为用 System Prompt，需要任务级定向用 DSP。

**DSP vs 角色扮演** — 角色扮演通过人格约束输出风格，DSP 通过关键词约束输出内容。角色扮演更适合风格控制，DSP 更适合内容聚焦。

## 模型差异

| 模型 | 表现 | 注意事项 |
|------|------|---------|
| Claude 4 | 对 hint 响应稳定，能准确理解关键词意图，不会过度字面化 | 过长的 hint 列表可能导致模型机械堆砌关键词而非真正理解方向 |
| GPT-4 | hint 效果稳定，对中英文关键词均有效 | 需要明确标记 hint（如方括号），否则可能当作正文处理 |
| Gemini 2 | 响应积极，有时会过度强调 hint 关键词 | 建议 hint 精简到 3-5 个词，过多会稀释效果 |
| 小型模型（<13B） | 效果不稳定，可能忽视 hint 或机械复制关键词 | 建议改用 few-shot 代替 hint |

## 常见踩坑

**hint 太模糊** — "关注重要内容"这样的 hint 没有方向性，等于没加。hint 要具体，"关注 Q3 财务数据和市场份额变化"才有价值。

**hint 与任务矛盾** — 任务要求客观分析，但 hint 给了主观立场。模型会陷入矛盾，输出质量下降。确保 hint 与任务目标一致。

**过度依赖 hint 代替清晰任务描述** — hint 是辅助引导，不是替代任务描述。先把任务说清楚，再用 hint 精调方向。

**hint 关键词太多** — 超过 5-7 个关键词后，模型注意力分散，各个关键词都覆盖到变成机械堆砌。精选最关键的 2-4 个。

**忘记验证 hint 方向是否正确** — 如果 hint 本身基于错误假设，模型会"高效地往错误方向走"。使用前先验证 hint 的合理性。

## 来源

- Li, Z. et al. (2023). *Guiding Large Language Models via Directional Stimulus Prompting*. NeurIPS 2023. https://arxiv.org/abs/2302.11520
- Prompt Engineering Guide (DAIR.AI). Directional Stimulus Prompting. https://www.promptingguide.ai/techniques/dsp
- 关联 wiki: [[wiki/prompt-system]]
