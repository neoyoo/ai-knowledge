---
pattern: few-shot-prompting
category: output
tags: [examples, in-context-learning, format-control, classification, agent-core]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: beginner
related_wiki: [wiki/prompt-system, wiki/context-management]
related_patterns: [chain-of-thought, zero-shot, structured-output]
---

# Few-Shot Prompting

> 在 prompt 中给出示例，通过"举一反三"引导模型学会任务模式和输出格式，无需微调就能让模型掌握新任务。

## 本质

想象你第一次让人做一件事：与其写三页说明书，不如给他看两个做好的样品——他立刻就懂了。Few-shot 对 LLM 做的是同一件事。

Few-shot 的核心机制是 **in-context learning**：模型在 forward pass 中从示例里提取隐含的模式——任务格式、标签空间、输出风格——然后把这个模式应用到新输入上。这不是重新训练，权重没有变化；改变的是模型在上下文窗口内"临时习得"的行为倾向。

一个关键发现（Min et al., 2022）：示例的标签是否正确，远没有示例的**格式**和**标签空间**重要。即使给出随机错误的标签，只要格式一致、标签类别正确，模型往往还是能预测对。这说明 few-shot 真正教给模型的是"任务的形状"，而不只是"正确答案"。

这个能力随模型规模显著增强（Kaplan et al., 2020；Touvron et al., 2023），在大模型上效果显著，在小模型上收益有限。

## 什么时候用

**输出格式需要稳定** — 当你需要模型严格遵守特定格式（JSON 字段顺序、特殊分隔符、固定前缀），用 few-shot 比在 system prompt 里反复描述格式更可靠。模型会直接模仿，而不是解读你的描述。

**分类和打标签任务** — 情感分析、意图识别、内容分级等任务，few-shot 能快速定义你的标签体系。特别是当你的标签定义和通用理解不一致时（如"客诉严重"在你的业务里有特殊边界），示例比定义更精确。

**风格模仿** — 品牌语调、特定作者风格、行业术语使用方式。三到五个示例的效果通常优于一段风格描述，因为风格本身难以言说，示例让模型直接"感受"。

**新概念或术语定义** — 当任务涉及领域专有词汇（如 "whatpu" 这类虚构词），zero-shot 模型无从推断；一个使用示例就够了。这是 few-shot 最干净的使用场景：1 个示例即可。

**自定义标签体系** — 当你有一套和通用预训练分布不同的分类维度，few-shot 比 fine-tuning 更轻量，比详细说明更准确。

## 什么时候不用

**示例太长吃 context** — 每个示例都会消耗 context window。如果单个示例本身就很长（如完整文档摘要对），3-shot 可能就占满了整个窗口，导致真正的任务输入被截断或降质。此时考虑 fine-tuning 或 RAG。

**任务足够直白（zero-shot 够用）** — "把这段话翻译成英文"不需要示例。过度使用 few-shot 会引入不必要的示例偏差，甚至让模型死板地模仿示例格式而忽略任务本身的灵活性。

**需要真正的推理而非模式匹配** — Few-shot 对需要多步推理的任务（数学、逻辑推断）帮助有限，模型可能只是模仿答案的格式而不是真的推导出答案。这类任务用 [[chain-of-thought]] 或 few-shot CoT。PEG 的验证数据：给出奇数求和是否为偶数的示例后，few-shot 仍然答错；加上推理步骤后才答对。

**需要泛化而非模仿** — 如果任务输入的多样性很高，而你的示例只覆盖了某几种类型，模型可能过拟合示例的特征，对差异较大的输入表现下降。

## 模板

### 基础版

**1-shot** — 最轻量的形式，适合定义新概念、展示格式：

```text
A "whatpu" is a small, furry animal native to Tanzania. An example of a sentence that uses the word whatpu is:
We were traveling in Africa and we saw these very cute whatpus.

To do a "farduddle" means to jump up and down really fast. An example of a sentence that uses the word farduddle is:
```

模型输出：

```text
When we won the game, we all started to farduddle in celebration.
```

一个示例就教会了模型"用新词造句"的任务格式。

---

**3-shot 分类** — 适合情感分析、意图识别等需要稳定标签的任务：

```text
分析以下评论的情感，输出 Positive 或 Negative。

评论：这款产品超出预期，电池续航特别棒！
情感：Positive

评论：客服态度很差，问题三天没解决。
情感：Negative

评论：包装很精美，但性价比一般。
情感：Negative

评论：{{input}}
情感：
```

关键点：示例数量足够定义标签边界，格式严格对齐，输入变量 `{{input}}` 放在最后一个示例的输入位置，让模型续写。

---

**示例选择策略** — 示例的质量比数量更重要：

```text
# 覆盖边界案例
示例要包含：
- 典型正例（清晰符合标签的）
- 典型负例（清晰不符合的）
- 边界案例（模糊、容易误判的）

# 均匀分布标签
如果有 3 个标签，每个标签至少 1 个示例；不要 4 个示例全是同一标签。

# 示例格式要一致
所有示例的格式必须完全相同——包括换行、冒号、空格位置。
模型对格式细节极度敏感。
```

---

### Agent 集成版

在 system prompt 中嵌入 few-shot 示例，用于行为校准——让 agent 的每次输出都遵循固定格式或决策风格：

```text
你是一个工单分流 agent。根据用户描述，判断工单类型并输出结构化结果。

示例：

用户：我的订单三天了还没发货，快递单号也查不到。
输出：
{
  "type": "logistics_delay",
  "priority": "high",
  "suggested_action": "escalate_to_fulfillment"
}

用户：APP 打开就闪退，重装了也没用。
输出：
{
  "type": "technical_bug",
  "priority": "medium",
  "suggested_action": "collect_device_info"
}

用户：我想修改收货地址，订单还没发货。
输出：
{
  "type": "order_modification",
  "priority": "low",
  "suggested_action": "redirect_to_self_service"
}

---

现在处理以下工单：

用户：{{ticket_content}}
输出：
```

注意：这里的 few-shot 起到双重作用——既定义了 JSON 字段结构，也隐式定义了 priority 和 action 的取值范围。比在 system prompt 里列举所有可能值更稳定。

## 变体

**1-shot / 3-shot / k-shot** — 示例数量是最直接的调节旋钮。经验规律：分类任务 3-5 shot 通常够用；复杂格式任务可能需要 5-10 shot；超过 10 shot 收益递减，且 context 消耗显著上升。最优数量要实验确定，不同任务差异很大。

**随机标签 few-shot（Min et al., 2022）** — 故意给示例打错误标签，验证模型是否真的在学任务形状而非死记答案。实验结论：标签正确与否对新模型影响不大，格式和标签空间才是关键。这个发现说明 few-shot 的核心价值在于"定义任务的形状"而非"提供正确案例"。

```text
# 随机标签示例（标签是错的，但格式是对的）
This is awesome! // Negative   ← 故意错误
This is bad! // Positive        ← 故意错误
Wow that movie was rad! // Positive
What a horrible show! //
```

模型仍然输出：`Negative`（正确），因为它理解了任务形式。

**动态 few-shot（RAG-based）** — 不硬编码示例，而是根据当前输入从示例库中检索语义最相近的 k 个示例。流程：构建示例向量索引 → 每次请求时用输入向量检索 top-k 示例 → 动态组装 prompt。适用于：示例库很大（数百到数千个）、输入分布变化大的生产系统。代价是增加一次向量检索延迟。

**Few-shot CoT** — few-shot 示例中不只给出输入-输出对，而是在每个示例里展示完整推理步骤。这是 few-shot 和 chain-of-thought 的组合，兼顾格式校准和推理质量。详见 [[chain-of-thought]]。

## 组合与选择

**Few-shot + CoT** → 格式校准 + 推理质量的最强组合。当 zero-shot CoT 的推理粒度不稳定，或你需要模型按特定推理风格（而非任意步骤）输出时，用 few-shot CoT。示例里同时展示推理步骤和最终答案，模型同时学会了"怎么想"和"怎么输出"。

**Few-shot + Structured Output** → 控制 JSON/XML 格式最可靠的方式。只用 schema 描述往往会有字段缺失、类型混用；加上 2-3 个完整 JSON 示例后，格式一致性显著提升。注意：示例的 JSON 必须严格合法，一个多余的逗号都会被模型模仿。

**Few-shot vs Zero-shot** → 决策规则很简单：先试 zero-shot，如果格式不稳定或任务需要定义特定标签/风格，再加 few-shot。不要把 few-shot 当成默认选项——它有 context 成本，而且可能引入不必要的约束。

**Few-shot vs Fine-tuning** → few-shot 是"借来的"能力，每次请求都要带上示例，有 context 成本；fine-tuning 是"买下的"能力，推理时不需要示例。判断标准：如果任务固定、示例集稳定、请求量大 → fine-tuning；如果任务多变、需要快速迭代示例 → few-shot。

## 模型差异

| 模型 | 表现描述 | 注意事项 |
|------|---------|---------|
| Claude 4 | few-shot 格式遵循度高，对示例格式细节（换行、缩进）的敏感度高；少量示例即可校准输出 | 示例中如果有格式不一致，Claude 会倾向于"智能平均"而非严格复制，可能造成格式漂移 |
| GPT-4 | few-shot 能力强，对随机标签的鲁棒性高（Min et al. 发现在 GPT 系列上尤为显著）；支持函数调用后 few-shot 的需求减少 | o1/o3 系列对 few-shot 的响应和标准 GPT-4 有差异，内置推理可能干扰纯格式模仿 |
| Gemini 2 | 长上下文窗口使得 k-shot（k 较大时）更可行；格式遵循稳定 | 示例过多时可能出现"注意力稀释"——靠近末尾的示例权重更高，建议把最典型的示例放在最后 |
| 开源模型（LLaMA 3、Qwen 2.5） | 7B 以下模型 few-shot 效果有限，常常只学到表面格式而非任务语义；70B+ 效果接近闭源 | 开源模型对示例顺序更敏感，最后一个示例对输出的影响比闭源模型大得多；建议用多种顺序测试后取中位数效果 |

## 常见踩坑

**示例偏差（示例分布不代表真实输入分布）** — 现象：模型在测试集上表现好，但生产环境遇到"示例没覆盖的类型"时输出乱套。原因：few-shot 示例隐式定义了任务范围，模型会往示例靠拢。避免方式：系统性地覆盖边界案例；用一批真实生产数据检验示例设计；在 prompt 末尾加"如果输入不属于以上任何类别，输出 unknown"。

**示例顺序影响结果** — 现象：把 3 个示例重新排列后，分类结果有变化，有时变化很大。原因：LLM 的注意力分布不均匀，后面的示例权重更高（"recency bias"），最后一个示例对模型的影响最显著。避免方式：把最有代表性的示例放在最后；对关键任务用多种顺序测试；动态 few-shot 时要意识到检索顺序即排列顺序。

**示例太多反而降效** — 现象：从 3-shot 增加到 10-shot，效果反而下降，输出变得不稳定。原因：过长的示例序列会稀释有效信号，模型开始在示例间寻找"共同规律"而非遵循每个示例。避免方式：控制在 3-8 shot 范围内；示例太多时考虑换 fine-tuning；如果确实需要覆盖更多类型，考虑动态 few-shot（按需检索）。

**标签分布不均导致偏向** — 现象：3 个示例都是 Positive，模型对模糊输入总倾向于输出 Positive。原因：示例的标签分布成为模型的先验。避免方式：尽量均匀分布标签；Min et al. 的研究表明"从真实标签分布采样"（而非均匀采样）效果更好，当真实数据不均衡时，少数类要刻意多放一些示例。

**格式细节被污染** — 现象：模型在输出中带上了示例里存在但不应该出现的字符（如示例里的 `//` 注释，模型输出时也加了 `//`）。原因：few-shot 的格式学习非常底层，任何字符都可能被学进去。避免方式：审查示例中每个非内容字符是否都是有意为之；用于分隔输入输出的符号要在所有示例中保持一致；最好用任务里不会自然出现的分隔符（如 `###`、`---`）。

## 来源

- Brown, T. et al. (2020). *Language Models are Few-Shot Learners*. NeurIPS 2020. https://arxiv.org/abs/2005.14165
- Min, S. et al. (2022). *Rethinking the Role of Demonstrations: What Makes In-Context Learning Work?* EMNLP 2022. https://arxiv.org/abs/2202.12837
- Kaplan, J. et al. (2020). *Scaling Laws for Neural Language Models*. https://arxiv.org/abs/2001.08361
- Touvron, H. et al. (2023). *LLaMA: Open and Efficient Foundation Language Models*. https://arxiv.org/pdf/2302.13971.pdf
- Prompt Engineering Guide (DAIR.AI). Few-Shot Prompting. https://www.promptingguide.ai/techniques/fewshot
- 关联 wiki: [[wiki/prompt-system]] [[wiki/context-management]]
