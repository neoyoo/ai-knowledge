---
template: evaluation-judge
scenario: LLM-as-Judge 评估其他模型输出
tags: [evaluation, judge, scoring, quality-assessment, llm-as-judge]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: advanced
related_patterns: [chain-of-thought, structured-output, few-shot]
---

# Evaluation Judge Template

> 用一个 LLM 作为评估器，对另一个 LLM 的输出打分或做对比，替代部分人工标注，实现自动化质量评估。

## 场景描述

在 LLM 应用开发中，评估模型输出质量是核心工程问题：人工标注成本高、周期长，传统指标（BLEU、ROUGE）对开放式文本相关性差。LLM-as-Judge 用一个强模型（通常是 GPT-4 或 Claude）作为评估器，按照预定义的 rubric 对目标模型的输出打分，在大规模评测、回归测试、AB 实验等场景中比人工标注快 10-100 倍，与人类判断的相关性在 0.7-0.9 之间。

## 模板

### System Prompt（单输出评分模式）

```text
你是一名严格、客观的 AI 输出质量评估专家。你的任务是按照提供的评分标准（Rubric），对 AI 的回答进行量化评分和定性分析。

## 评估原则

- **客观性**：只根据 Rubric 标准评分，不受回答长度、语气、格式影响
- **独立性**：不考虑哪个模型生成了这个答案
- **证据导向**：每个分数必须有具体的文本依据支撑
- **严格标准**：满分应该是真正优秀的回答，不要因为"还不错"就给高分

## 输出格式

必须严格按以下 JSON 格式输出：

\`\`\`json
{
  "scores": {
    "{{dimension_1}}": {
      "score": 0,
      "max_score": 0,
      "rationale": "给出此分数的具体理由，引用回答中的原文"
    }
  },
  "total_score": 0,
  "max_total_score": 0,
  "normalized_score": 0.0,
  "verdict": "EXCELLENT | GOOD | ACCEPTABLE | POOR | FAIL",
  "strengths": ["做得好的地方，具体说明"],
  "weaknesses": ["不足之处，具体说明"],
  "improvement_suggestion": "如果要改进，最关键的一个改动是什么"
}
\`\`\`

verdict 映射：
- EXCELLENT: normalized_score >= 0.9
- GOOD: 0.75 <= normalized_score < 0.9
- ACCEPTABLE: 0.6 <= normalized_score < 0.75
- POOR: 0.4 <= normalized_score < 0.6
- FAIL: normalized_score < 0.4
```

### System Prompt（A/B 对比评估模式）

```text
你是一名严格、客观的 AI 输出质量评估专家。你的任务是对同一问题的两个不同 AI 回答进行对比评估，判断哪个更好，或者它们是否持平。

## 评估原则

- **客观性**：只根据 Rubric 标准评判，不受回答顺序影响
- **Position Bias 防范**：先独立评估每个回答，再做对比，避免因"第一个看到的"就偏向它
- **细粒度分析**：在不同维度上分别判断哪个更好，不要只给总体结论
- **诚实**：如果两个回答质量相当，直接说 TIE，不要强行选出胜者

## 输出格式

\`\`\`json
{
  "dimension_comparisons": [
    {
      "dimension": "维度名称",
      "winner": "A | B | TIE",
      "explanation": "具体说明为什么 A/B 更好，或为什么持平"
    }
  ],
  "overall_winner": "A | B | TIE",
  "confidence": "HIGH | MEDIUM | LOW",
  "confidence_reason": "对整体判断的把握程度说明",
  "a_strengths": ["回答 A 的优点"],
  "b_strengths": ["回答 B 的优点"],
  "summary": "一段话总结对比结果"
}
\`\`\`
```

### User Prompt（单输出评分）

```text
## 问题

{{question_or_task}}

## AI 回答

{{model_output}}

## 评分标准（Rubric）

{{rubric_definition}}
```

### User Prompt（A/B 对比评估）

```text
## 问题

{{question_or_task}}

## 回答 A

{{output_a}}

## 回答 B

{{output_b}}

## 评分维度

{{dimensions_list}}

请先分别评估 A 和 B，再做对比判断。
```

### Rubric 定义模板

Rubric 是评分标准，直接影响评估质量。以下是通用 Rubric 模板，可根据场景替换维度：

```text
## 评分标准（Rubric）

### 准确性（Accuracy）— 满分 4 分
- 4 分：所有陈述完全正确，没有事实错误
- 3 分：主要内容正确，有 1 个可忽略的小错误
- 2 分：核心结论正确，但有明显的细节错误
- 1 分：部分正确，有重要的事实错误
- 0 分：主要内容错误或严重误导

### 完整性（Completeness）— 满分 3 分
- 3 分：完整回答了问题的所有方面
- 2 分：回答了主要方面，有次要内容遗漏
- 1 分：只回答了问题的一部分
- 0 分：基本没有回答问题

### 清晰度（Clarity）— 满分 3 分
- 3 分：表达清晰，结构合理，易于理解
- 2 分：大体清晰，有少量表达不清的地方
- 1 分：部分内容难以理解
- 0 分：表达混乱，难以理解

**总分：10 分**
```

## 自定义指南

**替换 Rubric 维度**：根据实际任务定制维度和权重。代码生成任务加"可运行性"和"安全性"维度；摘要任务加"信息保留度"和"无幻觉"维度；客服对话加"解决率"和"语气"维度。

**减少 Position Bias**（A/B 对比时的关键）：
1. 在 system prompt 中明确要求"先独立评估再对比"
2. 对同一组对比做两次评估，第二次交换 A/B 顺序，如果结论相反则标记为不确定
3. 在 user prompt 末尾加"注意：不要因为某个回答先出现就偏向它"

**减少 Verbosity Bias**（模型偏好长回答）：在 system prompt 中加"不要因为回答更长就给更高分。简洁准确的短回答可能优于啰嗦的长回答"，并在 Rubric 的清晰度维度中明确"冗余内容会扣分"。

**多评估员聚合**：用不同 temperature 对同一任务运行 3-5 次，取均值或多数投票。也可以用两个不同的模型分别评估，如果结论差异大则送人工复核。

**标定（Calibration）**：在正式评估前，用 5-10 个已知质量的样本测试 rubric，确认模型打分和你的预期一致。如果系统性偏高/偏低，在 system prompt 中调整描述（如"4 分是很难达到的满分，大多数好回答应该是 3 分"）。

## 使用示例

**输入**（单输出评分，客服对话场景）：

```
问题：用户询问"我的订单什么时候到货"

AI 回答：
"您好！感谢您的咨询。根据您的订单信息，预计送达时间为 3-5 个工作日。
如果您需要了解实时物流信息，可以在我们的 App 中点击'我的订单'→'查看物流'查看最新状态。
如有其他问题，欢迎随时联系我们！"

评分标准（Rubric）：
- 问题解决度（4分）：是否回答了用户的核心问题
- 信息准确性（3分）：提供的信息是否正确（预计送达时间是已知的 3-5 工作日）
- 服务态度（2分）：语气是否友好专业
- 引导操作（1分）：是否提供了自助操作路径
```

**输出**（示意）：

```json
{
  "scores": {
    "问题解决度": {
      "score": 3,
      "max_score": 4,
      "rationale": "直接给出了 3-5 工作日的答复，解决了核心问题，但没有提及具体下单时间可能影响送达，略显泛化"
    },
    "信息准确性": {
      "score": 3,
      "max_score": 3,
      "rationale": "3-5 工作日的信息与 Rubric 中标注的已知信息一致，无错误"
    },
    "服务态度": {
      "score": 2,
      "max_score": 2,
      "rationale": "语气友好，开头有问候，结尾有邀请，专业度合适"
    },
    "引导操作": {
      "score": 1,
      "max_score": 1,
      "rationale": "明确给出了 App 内查看物流的操作路径，步骤清晰"
    }
  },
  "total_score": 9,
  "max_total_score": 10,
  "normalized_score": 0.9,
  "verdict": "EXCELLENT",
  "strengths": ["直接回答了问题，没有绕弯子", "提供了具体的自助操作路径"],
  "weaknesses": ["未考虑下单时间对到货时间的影响，回答略有泛化"],
  "improvement_suggestion": "加一句'根据您的下单时间，预计[具体日期]前送达'会更精准"
}
```

## 适配建议

**选择合适的 judge 模型**：用比被评估模型更强的模型做 judge（如用 Claude Opus 评估 Haiku 的输出）。用同等能力的模型互评时，偏差较大；用更弱的模型做 judge 则完全不可信。

**单输出 vs A/B 对比的选择**：单输出评分适合绝对质量评估、大批量回归测试；A/B 对比适合模型选型、prompt 版本比较。A/B 对比的人类相关性通常比单输出评分高，因为相对判断比绝对打分更符合人类直觉。

**处理主观性强的任务**：对于"创意写作好不好"这类主观任务，rubric 中要明确评估的是客观维度（如"是否回应了用户的具体要求""是否有明显语法错误"），而不是"整体风格是否好"。主观判断需要人工标注。

**大批量评估的成本控制**：对于 1000+ 条的评估，先用小模型（如 Haiku）做初筛，只把得分在边界值附近（如 normalized_score 0.5-0.7）的样本送给强模型复核，可降低 60-70% 的成本。

**验证评估器可信度**：从评估集中抽取 10% 送人工标注，计算模型打分与人工打分的 Spearman 相关系数。低于 0.6 说明 rubric 或 judge prompt 需要调整；0.7-0.85 是可用范围；0.85+ 可基本信任自动评估。

## 关联

- 模式：[[cookbook/prompts/patterns/chain-of-thought]] — judge 的评分推理是 CoT 的直接应用，rationale 字段本质是对打分的 CoT
- 模式：[[cookbook/prompts/patterns/structured-output]] — 评估结果必须是结构化 JSON，便于批量处理和统计
- 模式：[[cookbook/prompts/patterns/few-shot]] — 在 user prompt 中加入已标注的示例（golden examples）可以显著提升评估一致性
- Wiki：[[wiki/evaluation]] — LLM-as-Judge 的理论背景、与人工标注的对比、偏差类型（position bias、verbosity bias、self-enhancement bias）
