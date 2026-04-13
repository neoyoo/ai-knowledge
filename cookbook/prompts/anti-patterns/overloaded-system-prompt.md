---
anti-pattern: overloaded-system-prompt
severity: warning
tags: [system-prompt, length, token-waste, compliance-degradation]
related_patterns: [system-prompt-design, prompt-chaining]
---

# Overloaded System Prompt

> 把所有规则、背景、示例塞进一个超长 system prompt，模型开始忽略中间段，遵循率随长度下降。

## 症状

- system prompt 超过 2000 tokens，还在不断增长
- 越来越多的规则加进去，但行为改善越来越有限
- 模型"遵守"了 prompt 开头的规则，但忘记了中间或末尾的规则
- 同样的规则，放在 system prompt 末尾比放在开头效果差很多
- 为了让某条规则生效，你把它加了三遍——有时管用，有时不管用

## 错误示例

```text
[System Prompt — 实际案例，约 3500 tokens]

你是 AcmeCorp 的智能客服助手，名叫 Aria。

# 关于你自己
你是一个友好、专业、耐心的助手。你热爱帮助用户解决问题。你对 AcmeCorp 的产品非常熟悉。你会用清晰简洁的语言回答问题。你不会使用复杂的技术术语，除非用户是技术专家。你会在适当的时候表达同理心。你会在回答结束时询问是否还有其他问题。

# 产品知识
AcmeCorp 有三款主要产品：BasicPlan（¥29/月）、ProPlan（¥99/月）、EnterpriseПлан（联系销售）。BasicPlan 包含功能 A、B、C。ProPlan 包含功能 A、B、C、D、E、F。Enterprise 包含全部功能加定制化服务。退款政策：购买后 7 天内可全额退款，7-30 天内可退 50%，30 天后不退。取消订阅：可随时取消，下个计费周期生效。数据导出：ProPlan 及以上支持导出 CSV 和 Excel，BasicPlan 不支持。API 访问：仅 EnterpriseПlan 支持。两步验证：所有方案均支持。...（更多产品细节，约 800 tokens）

# 回复规范
- 回复要简洁，不超过 200 字
- 专业术语要解释
- 使用项目符号列表展示多个选项
- 回复末尾附上相关文档链接（如果有的话）
- 不要使用表情符号
- 中文回复
- 标点符号使用中文全角
- 不要用"您好"开头，直接进入正题
- 数字要用阿拉伯数字，不要用汉字数字
- ...（更多格式规则，约 300 tokens）

# 禁止事项
- 不要讨论竞争对手
- 不要承诺产品路线图或未发布功能
- 不要给出具体的法律建议
- 不要泄露内部系统信息
- ...（更多禁止项，约 200 tokens）

# 升级流程
遇到以下情况时转人工：账单争议超过 ¥500、用户明确要求转人工、三轮对话未能解决问题、用户情绪激动...（约 200 tokens）

# 示例对话
[三个完整的 Q&A 示例，共约 600 tokens]
```

这个 system prompt 约 3500 tokens。研究和实践均表明，模型在处理超长 system prompt 时存在"中间遗忘"现象：靠近开头和结尾的内容被记住，中间段内容遵循率明显下降。

## 为什么有害

**注意力稀释**：Transformer 的 attention 机制对长序列中的中间位置存在天然偏弱效应（Lost in the Middle 研究，Liu et al. 2023）。超过约 2000 tokens 的 system prompt 里，中间位置的规则会被"淡化"。

**规则优先级混乱**：规则太多时，模型无法判断哪条更重要。遇到边缘情况时，它会选择一条看起来最相关的执行，忽略其他——哪条被选中是随机的。

**token 浪费**：每次调用都携带完整 system prompt，但其中大量内容在当前对话里根本不相关。一个询问退款的用户不需要看到 API 访问和数据导出的规则。

**维护负担**：超长 prompt 变成"人人往里加、没人敢删"的规则垃圾场。每次修改都可能破坏之前已经调好的行为，形成脆弱性不断累积的技术债。

## 正确做法

**策略 A：分层 prompt（最重要的改变）**

把"始终有效的核心规定"和"按场景动态注入的上下文"分开：

```text
[System Prompt — 精简到 ~400 tokens]

你是 AcmeCorp 的客服助手 Aria。

核心规则（不可违反）：
- 不讨论竞争对手
- 不承诺未发布功能
- 账单争议超过 ¥500 或用户要求时转人工

回复风格：简洁（≤150字）、中文、阿拉伯数字、不用"您好"开头

当前用户套餐：{{user_plan}}
```

然后在每次调用时，根据用户意图动态注入相关的产品知识片段：

```text
[动态注入 — 仅在用户询问退款时添加]

退款政策：
- 7 天内：全额退款
- 7-30 天：退 50%
- 30 天后：不退
```

**策略 B：把示例移出 system prompt**

few-shot 示例放在 system prompt 里会大量消耗 token 预算。改用 user/assistant 消息对的形式传入，或仅在特定场景下触发：

```text
# 调用结构（伪代码）
messages = [
    {"role": "system", "content": CORE_SYSTEM_PROMPT},        # 400 tokens
    {"role": "user", "content": FEW_SHOT_EXAMPLE_1_Q},        # 按需插入
    {"role": "assistant", "content": FEW_SHOT_EXAMPLE_1_A},
    {"role": "user", "content": actual_user_message}
]
```

**策略 C：重要规则放首尾，不要堆中间**

如果 system prompt 确实需要一定长度，把最关键的规则放在开头（第一段）和结尾（最后一段），中间放背景信息：

```text
[开头] 核心禁止项 + 角色定义
[中间] 产品知识（参考性内容，偶尔查用）
[结尾] 回复格式规则（每次都需要遵守）
```

**长度参考基准：**

| system prompt 长度 | 遵循率 | 建议 |
|-------------------|--------|------|
| < 500 tokens | 高 | 理想范围 |
| 500-1500 tokens | 较高 | 可接受 |
| 1500-3000 tokens | 中等，中间段开始衰减 | 考虑分层 |
| > 3000 tokens | 明显下降 | 必须重构 |

## 修复检查清单

- [ ] system prompt 当前有多少 tokens？是否超过 1500？
- [ ] 其中有哪些内容是"仅在特定场景下才相关"的？可以做动态注入
- [ ] few-shot 示例是否可以移到 user/assistant 消息对，而不占用 system prompt 空间？
- [ ] 能否提炼出 5-10 条真正"始终有效"的核心规则，其余的按需加载？
- [ ] 规则是否有重复表述（同一条规则用不同的话说了两次）？
- [ ] 最关键的格式规则是否在 system prompt 的开头或结尾，而不是埋在中间？
- [ ] 是否定期清理已无效或已被其他规则覆盖的旧规则？

## 关联

- [[cookbook/prompts/patterns/system-prompt-design]] — system prompt 的完整设计方法，包括结构化分层和模块化管理
- [[cookbook/prompts/patterns/prompt-chaining]] — 把长流程拆成多个短调用，每步 prompt 保持精简，而不是把全部逻辑塞进一个 prompt
