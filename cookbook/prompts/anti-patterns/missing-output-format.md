---
anti-pattern: missing-output-format
severity: warning
tags: [output, format, parsing, consistency]
related_patterns: [structured-output, few-shot, system-prompt-design]
---

# Missing Output Format

> 没有告诉模型输出应该长什么样，程序无法可靠解析，人工处理成本剧增。

## 症状

- 同一个 prompt 有时输出 JSON，有时输出 Markdown，有时输出纯文本
- 你的解析代码需要大量 try/except 来处理各种输出变体
- 模型输出了正确的内容，但夹在一段解释文字中间，你得手动提取
- 字段名、大小写、嵌套结构在不同次调用之间不一致（`user_name` vs `userName` vs `name`）
- 输出里出现了你没要求的 Markdown 标题或代码围栏，干扰了下游处理

## 错误示例

```text
分析以下用户评论的情感，并提取关键信息。

评论：这款耳机音质很好，但佩戴久了会有点不舒服，而且价格偏贵。
```

模型可能输出以下任意一种形式（在不同次调用中随机变化）：

**变体 A（散文）：**
```
该评论整体情感偏中性，用户对音质表示满意，但对佩戴舒适度和价格持有保留意见。主要提及点：音质（正面）、佩戴舒适度（负面）、价格（负面）。
```

**变体 B（Markdown 列表）：**
```
## 情感分析结果

- **整体情感**：中性偏负
- **正面方面**：音质好
- **负面方面**：佩戴不舒适、价格偏贵
```

**变体 C（JSON，但结构不稳定）：**
```json
{
  "sentiment": "neutral",
  "positives": ["音质好"],
  "negatives": ["佩戴不适", "价格贵"]
}
```

三种输出都包含正确信息，但没有一种格式是可预测的。写一个能可靠处理这三种输出的解析器，比直接约束输出格式要复杂 10 倍。

## 为什么有害

模型在没有格式约束时会根据内容语义"自由发挥"——它认为什么格式最适合表达这个内容就用什么格式。这个判断通常是合理的，但对于需要程序消费的场景，"合理"远远不够，你需要的是"可预测"。

后果分级：

**轻微**：人工审阅时需要多花时间格式化输出。

**中等**：解析代码需要大量容错逻辑，维护成本高，边缘情况不断涌现。

**严重**：在 agent pipeline 里，格式不稳定会导致工具调用失败、状态更新丢失、下游 agent 收到损坏的输入——整个 pipeline 的可靠性被一个没有格式约束的节点拖垮。

## 正确做法

**方案 A：直接在 prompt 中指定格式**

```text
分析以下用户评论，严格按照下方 JSON schema 输出，不要包含任何其他文字。

Schema：
{
  "overall_sentiment": "positive" | "negative" | "neutral" | "mixed",
  "score": -1.0 到 1.0 之间的浮点数（-1 最负，1 最正）,
  "aspects": [
    {
      "aspect": string,          // 评价维度（如"音质"、"价格"）
      "sentiment": "positive" | "negative" | "neutral",
      "quote": string            // 评论中支持这个判断的原文片段
    }
  ]
}

评论：这款耳机音质很好，但佩戴久了会有点不舒服，而且价格偏贵。
```

输出将稳定为：
```json
{
  "overall_sentiment": "mixed",
  "score": -0.2,
  "aspects": [
    {"aspect": "音质", "sentiment": "positive", "quote": "音质很好"},
    {"aspect": "佩戴舒适度", "sentiment": "negative", "quote": "佩戴久了会有点不舒服"},
    {"aspect": "价格", "sentiment": "negative", "quote": "价格偏贵"}
  ]
}
```

**方案 B：使用 few-shot 示例锁定格式**

当 schema 描述比较复杂时，一个示例胜过千字说明：

```text
分析用户评论情感。按示例格式输出。

示例输入：电池续航超长，就是充电速度慢了点。
示例输出：
SENTIMENT: mixed
ASPECTS:
  + 电池续航: 正面（"续航超长"）
  - 充电速度: 负面（"充电速度慢"）

现在分析：这款耳机音质很好，但佩戴久了会有点不舒服，而且价格偏贵。
```

**方案 C：使用 structured output API（最稳定）**

对于支持的模型（Claude、GPT-4 等），直接传入 JSON Schema 作为 response_format 参数，从 API 层面强制约束输出结构。不需要在 prompt 里解释格式，解析零容错处理。

适合生产环境中对稳定性要求最高的场景。详见 [[cookbook/prompts/patterns/structured-output]]。

## 修复检查清单

- [ ] 如果输出需要被程序消费，是否指定了精确的格式（JSON/XML/特定分隔符格式）？
- [ ] 是否明确告知模型"不要输出格式以外的内容"（如解释、前言、后记）？
- [ ] JSON 字段名的大小写规范是否统一并在 prompt 中指定？
- [ ] 对于枚举值（如情感类型），是否列出了所有允许的值？
- [ ] 是否考虑使用 few-shot 示例来强化格式要求，而不只是文字描述？
- [ ] 对于高稳定性要求的场景，是否评估了 structured output API？
- [ ] 在 system prompt 中是否有持久的格式基础规范（如"所有输出必须是合法 JSON"）？

## 关联

- [[cookbook/prompts/patterns/structured-output]] — 格式约束的完整方案：JSON Schema、few-shot 格式锁定、API 层强制约束的使用时机
- [[cookbook/prompts/patterns/few-shot]] — 用示例传达格式期望，是比文字描述更可靠的方式
- [[cookbook/prompts/patterns/system-prompt-design]] — 在 system prompt 中建立全局格式基准，避免每个 user prompt 都重复指定
