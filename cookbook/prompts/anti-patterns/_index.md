# Prompt Anti-patterns

> 常见提示词设计错误和避坑指南。每个 anti-pattern 包含症状、错误示例、正确做法和修复检查清单。

## 按严重度排列

### Critical（必须修复）
- [Vague Instructions](vague-instructions.md) — 指令模糊不清，模型只能猜测你的意图，每次输出都不一样。
- [Contradictory Rules](contradictory-rules.md) — 同一个 prompt 里的规则互相打架，模型陷入无法同时满足的困境，只能随机选边站。
- [Prompt Injection Vulnerability](prompt-injection-vulnerability.md) — 用户输入未经隔离直接拼入 prompt，攻击者可通过构造恶意输入覆盖系统指令，让模型执行未授权行为。

### Warning（建议修复）
- [Missing Output Format](missing-output-format.md) — 没有告诉模型输出应该长什么样，程序无法可靠解析，人工处理成本剧增。
- [Overloaded System Prompt](overloaded-system-prompt.md) — 把所有规则、背景、示例塞进一个超长 system prompt，模型开始忽略中间段，遵循率随长度下降。
- [Hardcoded Examples](hardcoded-examples.md) — Few-shot 示例硬编码且从不更新，导致模型过度模仿示例的表面特征，在真实输入分布上系统性偏差。
- [Ignoring Model Differences](ignoring-model-differences.md) — 在一个模型上调好的 prompt 直接用于另一个模型，忽视不同模型的训练偏好、指令遵循风格和能力边界，导致效果悄然下降却难以排查。

## 快速自查
写完 prompt 后过一遍这个清单：
1. 指令是否具体？（→ vague-instructions）
2. 规则之间有没有矛盾？（→ contradictory-rules）
3. 输出格式有没有明确约束？（→ missing-output-format）
4. System prompt 是不是太长了？（→ overloaded-system-prompt）
5. 用户输入有没有做隔离？（→ prompt-injection-vulnerability）
6. Few-shot 示例是否多样且均衡？（→ hardcoded-examples）
7. 换了模型还能用吗？（→ ignoring-model-differences）
