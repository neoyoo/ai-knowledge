---
pattern: reflexion
category: action
tags: [self-critique, retry, error-recovery, learning-from-failure, agent-improvement]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: advanced
related_wiki: [wiki/query-loop, wiki/evaluation-observability]
related_patterns: [react, chain-of-thought, self-consistency]
---

# Reflexion

> ReAct 的升级版——在行动循环中加入自我反思层，让模型从失败中提炼教训，带着教训重试，而不是盲目重来。

## 本质

人犯了错会复盘："哪里做错了？下次怎么改？"然后带着这个认知重新出发。普通重试没有这一步——只是把同样的错误再犯一遍。

Reflexion 把这个"复盘-改进"过程形式化给了 LLM：

- **Act**：执行任务（ReAct 循环，Thought + Action + Observation）
- **Evaluate**：判断这次尝试成功了吗？（可以是 LLM 打分、规则判断、测试用例执行）
- **Reflect**：失败了的话，总结这次的教训——"我犯了什么错，下次应该换什么策略"
- **Retry**：把反思内容存入记忆，开始下一轮尝试，这次带着上一轮的教训

与纯 ReAct 的本质区别：ReAct 的每轮循环只利用当前 episode 的 Observation。Reflexion 跨 episode 积累教训，每次失败后都变得更聪明一点。

三个核心角色（Shinn et al. 2023 原论文）：

- **Actor**：执行动作的 LLM，使用 ReAct 或 CoT 驱动，带有短期记忆（当前 episode 轨迹）
- **Evaluator**：评估这次尝试的质量，输出奖励信号（成功/失败/得分）
- **Self-Reflection**：根据奖励信号 + 当前轨迹 + 历史经验，生成具体的语言反思，存入长期记忆

## 什么时候用

- **复杂任务首次成功率不高**：代码生成、多步规划、工具使用任务——一次成功率 50-80%，加 Reflexion 后迭代几轮能大幅提升
- **有明确的成功判断标准**：单元测试能跑通、答案精确匹配、结构验证通过——这是 Evaluate 步骤的基础，没有评判标准 Reflexion 无法运作
- **可以接受多轮延迟**：每次失败都多一轮 LLM 调用（Reflect）+ 下一轮重试，适合离线批处理、代码生成、研究任务，不适合实时交互
- **代码生成 + 测试驱动场景**：写代码 → 跑测试 → 测试失败 → 反思失败原因 → 重写代码，这是 Reflexion 最经典的应用场景
- **决策任务需要策略演进**：游戏、规划、工具序列选择——从多次失败中归纳策略模式

## 什么时候不用

- **简单任务，一次就对**：格式转换、简单翻译、单步查询——引入反思循环只增加延迟，没有收益
- **没有评判标准**：开放式创作、没有"对错"的任务——Evaluator 无法工作，Reflexion 就是空壳
- **延迟不允许重试**：实时对话、毫秒级响应要求——每次失败都多一轮推理，时间代价太高
- **反思质量无法保证**：模型能力弱、任务超出模型理解范围——生成的反思是错的，越反思越跑偏
- **Context 已经很紧张**：历史反思内容存入 context 会持续消耗 token，长任务 + 多次失败 + 浅 context 的组合很危险

## 模板

### 基础版（单轮反思）

最小实现：一次失败后反思，带着反思重试。适合理解概念和快速原型。

```text
## 第一轮尝试

任务：写一个 Python 函数，接受字符串列表，返回去重后按字母排序的结果。

[模型输出]
def deduplicate_and_sort(items):
    return sorted(set(items))

## 评估

运行测试：
✗ test_empty_list: 通过
✗ test_case_insensitive: 失败 — 预期 ['apple', 'Banana'] 视为重复，实际返回两个元素
✗ test_preserves_original_case: 失败

## 自我反思

上一次尝试的错误：我没有理解"去重"是大小写不敏感的。直接用 set() 会把 'apple' 和 'Apple' 视为不同元素。
需要改进的策略：去重时转小写比较，但保留原始大小写。可以用字典保留第一次出现的原始形式。

## 第二轮尝试（带教训）

[模型输出，参考上方反思]
def deduplicate_and_sort(items):
    seen = {}
    for item in items:
        key = item.lower()
        if key not in seen:
            seen[key] = item
    return sorted(seen.values(), key=str.lower)
```

### Agent 集成版（带记忆的反思循环）

生产环境的 Reflexion 形态：将反思内容持久化到 memory，跨 episode 积累教训，每轮尝试都能利用所有历史反思。

**System Prompt 模板（带反思记忆）：**

```text
你是一个具备自我改进能力的任务执行助手。

## 工作模式

每次执行任务前，你会收到：
1. 任务描述
2. 历史反思记忆（如果有）：你在之前的尝试中总结的教训

## 执行规则

1. 先读取历史反思记忆，理解之前失败的原因和改进策略
2. 基于教训制定当前方案，避免重蹈覆辙
3. 执行任务，使用可用工具
4. 每次尝试都要完整执行，不要半途而废

## 反思格式（任务失败时使用）

当评估结果为失败时，输出以下格式的反思：

**失败原因**：[具体说明哪里做错了，要精确，不要泛泛而谈]
**错误类型**：[逻辑错误 / 工具使用错误 / 理解偏差 / 边界情况遗漏 / 其他]
**改进策略**：[下次应该怎么做，给出具体可执行的改变]
**不要重复的行为**：[明确列出下次绝对不能再做的事]

## 历史反思记忆

{reflection_memory}
（由框架在每轮尝试前注入，初次为空）
```

**Reflexion Agent 循环骨架（Python）：**

```python
import anthropic

client = anthropic.Anthropic()

def run_reflexion_agent(
    task: str,
    evaluator,           # callable: (attempt_result) -> (success: bool, feedback: str)
    max_attempts: int = 5
) -> dict:
    reflection_memory = []   # 跨 episode 的长期记忆
    
    for attempt in range(1, max_attempts + 1):
        # 构造带有反思记忆的 prompt
        memory_text = "\n\n".join(reflection_memory) if reflection_memory else "（无历史记录，这是第一次尝试）"
        
        system = SYSTEM_PROMPT.replace("{reflection_memory}", memory_text)
        messages = [{"role": "user", "content": task}]
        
        # 执行当前 episode（可以是 ReAct 循环）
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=system,
            messages=messages
        )
        attempt_result = response.content[0].text
        
        # 评估：成功了吗？
        success, feedback = evaluator(attempt_result)
        
        if success:
            return {
                "success": True,
                "result": attempt_result,
                "attempts": attempt,
                "reflections": reflection_memory
            }
        
        # 失败：生成自我反思，存入长期记忆
        reflection = generate_reflection(
            task=task,
            attempt_result=attempt_result,
            feedback=feedback,
            attempt_num=attempt
        )
        reflection_memory.append(f"[第 {attempt} 次尝试的教训]\n{reflection}")
    
    return {
        "success": False,
        "result": attempt_result,
        "attempts": max_attempts,
        "reflections": reflection_memory
    }


def generate_reflection(task, attempt_result, feedback, attempt_num) -> str:
    """调用 LLM 生成语言化的自我反思。"""
    prompt = f"""任务：{task}

第 {attempt_num} 次尝试的结果：
{attempt_result}

评估反馈：
{feedback}

请按以下格式总结这次失败的教训：
**失败原因**：
**错误类型**：
**改进策略**：
**不要重复的行为**："""
    
    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=512,
        messages=[{"role": "user", "content": prompt}]
    )
    return response.content[0].text


# 使用示例：代码生成 + 测试驱动
def code_evaluator(code: str) -> tuple[bool, str]:
    """运行测试用例，返回成功标志和反馈。"""
    import subprocess
    test_script = f"{code}\n\n{TEST_CASES}"  # 注入测试用例
    result = subprocess.run(
        ["python", "-c", test_script],
        capture_output=True, text=True, timeout=10
    )
    if result.returncode == 0:
        return True, "所有测试通过"
    else:
        return False, f"测试失败：\n{result.stderr}"

result = run_reflexion_agent(
    task="写一个函数：给定整数列表，返回第二大的数。处理重复元素和空列表边界情况。",
    evaluator=code_evaluator,
    max_attempts=4
)
```

**关键设计点：**

1. **反思要存入长期记忆，不是短期历史**：反思内容不应该只是追加到对话历史末尾（那会很快撑爆 context），而是提炼成精简的教训注入到下一轮的 system prompt 或 user prompt 开头。

2. **Evaluator 设计是成败关键**：代码用测试用例（确定性强）；问答用 LLM 打分（需要评分 prompt 够精准）；规划用规则检查（结构验证）。Evaluator 的质量上限决定了 Reflexion 的质量上限。

3. **反思精度比次数重要**：生成"我做错了"没用，要生成"我在处理 Unicode 边界时没有转义，下次必须先对输入做 `.encode('utf-8')` 处理"这样具体可执行的教训。

## 变体

| 变体 | 核心机制 | 适用场景 | 主要取舍 |
|------|----------|----------|----------|
| **单轮反思** | 一次失败 → 反思 → 重试，最多 2 次 | 简单任务、latency 敏感 | 实现简单，但改进空间有限 |
| **多轮迭代** | 持续反思直到通过或达到上限 | 代码生成、复杂规划 | 效果好，token 消耗随失败次数线性增长 |
| **带外部反馈** | Evaluator 是真实测试用例 / 编译器 / 人工评分 | 编程任务、结构化验证 | 评估最准确，但需要外部工具支持 |
| **CoT + Reflexion** | ReAct 改为 CoT 作为 Actor，其余相同 | 推理密集型任务、不需要工具 | 减少工具调用开销，但失去实时信息获取 |
| **跨任务记忆** | 反思记忆跨不同任务持久化（存数据库） | 重复类型的任务（如批量代码生成） | 能从历史任务中学习，但记忆管理复杂度高 |

**选择建议：**
- 有测试用例 → 带外部反馈变体（最准确）
- 没有测试但有明确标准 → 多轮迭代 + LLM 评估
- latency 有要求 → 单轮反思（失败率高时做一次，不多做）
- 同类任务批量处理 → 跨任务记忆（积累经验后越来越好）

## 组合与选择

**Reflexion + ReAct**

Reflexion 以 ReAct 作为 Actor 层，是最常见的组合。ReAct 处理单个 episode 内的动态工具调用，Reflexion 处理 episode 间的经验积累。两者分工明确：ReAct 管"这一轮怎么做"，Reflexion 管"下一轮怎么改"。

**Reflexion + CoT**

Actor 用 CoT 替代 ReAct，适合不需要外部工具的纯推理任务。论文中 HotPotQA 实验用的就是 CoT + Reflexion，比纯 CoT 显著提升多跳推理准确率。

**Reflexion + Self-Consistency**

Self-Consistency 是"采样多个答案取多数"，Reflexion 是"反思失败迭代改进"。两者方向不同：前者靠广度（多样性），后者靠深度（改进）。适合高精度要求的场景可以组合：先 Reflexion 迭代到最优，再对最后结果做 Self-Consistency 验证。

**Reflexion vs 纯 ReAct**

| 维度 | 纯 ReAct | Reflexion |
|------|----------|-----------|
| 失败处理 | 在当前 episode 内调整 | 跨 episode 积累教训 |
| 首次失败后 | 可能重复同样错误 | 反思教训，改变策略 |
| Token 消耗 | 仅当前 episode | 每次失败多一次反思调用 |
| 适用场景 | 单次成功率较高的任务 | 需要多次尝试的复杂任务 |

结论：**首次成功率够高就用 ReAct，任务难、首次失败率高就加 Reflexion 层**。

## 模型差异

| 模型 | 反思质量 | 遵循历史教训 | 自我评估准确性 | 注意事项 |
|------|----------|--------------|----------------|----------|
| **Claude 4** (Sonnet/Opus) | 反思具体、可操作性强 | 对记忆中的教训遵循度高 | 倾向于承认错误，不过度辩护 | Opus 反思质量明显优于 Sonnet；偶尔反思过长，建议限制 max_tokens |
| **GPT-4o** | 反思结构化，条理清晰 | 遵循度好，但偶尔会"理解但不执行" | 自我评估偏乐观，容易高估当前方案 | 建议在 Evaluator prompt 中加强客观标准，减少模型自我评估偏差 |
| **Gemini 2** | 反思质量中等 | 对记忆注入的响应稳定 | 自我评估一致性较好 | 长上下文下反思历史注入效果稳定；但反思内容有时过于简短，需要 prompt 引导深度 |
| **开源模型** | 差异极大，依赖微调 | 小参数模型容易忽略注入的教训 | 自我评估能力弱，可能生成无效反思 | 建议增加反思格式约束（JSON schema）；避免反思内容太长导致 context 压力 |

**工程建议**：无论哪个模型，都要为反思生成单独设置较低的 `max_tokens`（256-512 即可）。反思要的是精准，不是长篇大论。

## 常见踩坑

**1. 反思太泛泛，没有可操作性**

症状：模型反思说"我需要更仔细地思考"、"我应该更加小心"，但下一轮还是犯同样的错。

原因：反思 prompt 没有要求具体的错误定位和具体的改进行动。

解决方案：
- 反思 prompt 里明确要求："说明具体是哪一行代码/哪个推理步骤出了问题"
- 要求反思必须包含"不要重复的行为"一栏，迫使模型做具体归纳
- 用结构化格式约束输出（JSON 或固定 section 标题），避免泛泛而谈

**2. 死循环（每次犯同样的错）**

症状：连续 3 轮失败，失败原因完全相同，反思内容也高度重复，没有实质进展。

原因：要么任务超出了模型能力边界，要么 Evaluator 的反馈信息不够具体，模型不知道"哪里"错了。

解决方案：
- 检测重复失败：连续 2 次相同失败原因 → 换策略提示（"你已经在同一个地方失败了两次，请彻底重新思考你的方案"）
- 丰富 Evaluator 反馈：不只返回"失败"，要返回具体的失败点（哪个测试用例、哪行报错、期望值 vs 实际值）
- 设置尝试上限并优雅退出，不要无限循环

**3. 反思历史撑爆 Context**

症状：多次失败后，注入 context 的反思历史越来越长，token 消耗暴涨，最终超出 context window。

原因：直接把所有反思原文拼接注入，没有压缩和管理。

解决方案：
- 限制保留的反思条数（滑动窗口，只保留最近 3-5 条）
- 对历史反思做摘要压缩（"前几次的综合教训是…"）
- 只保留"不要重复的行为"清单，丢弃失败过程的详细描述

**4. Evaluator 本身不准**

症状：Reflexion 循环了 4 轮，最终结果其实是错的，但 Evaluator 说通过了。或者：正确答案被 Evaluator 判为失败，导致无意义的重试。

原因：用 LLM 做 Evaluator 时，LLM 评估本身有幻觉和偏差。

解决方案：
- 能用确定性评估（测试用例、正则、结构验证）就不用 LLM 评估
- LLM 评估时给 Evaluator 提供评分标准和反例，减少主观判断空间
- 对评估结果做校验：同一结果用两个不同 prompt 独立评估，结果不一致时标记为"不确定"

**5. 反思存入记忆但 Actor 不看**

症状：历史反思被正确注入到 prompt，但模型在新一轮直接忽略了历史教训，又做了同样的事。

原因：反思内容在 prompt 里位置不对，或 system prompt 没有明确指示"先读记忆再行动"。

解决方案：
- 反思记忆放在 prompt 靠前位置（system prompt 的结尾，而不是 user message 的末尾）
- 显式指令："在开始执行之前，请先读取历史反思记忆，说明你将如何避免上次的错误"
- 要求模型在第一个 Thought 里先回顾记忆（强制激活注意力）

## 来源

- **Shinn et al. (2023)** — Reflexion 原论文，提出 Actor/Evaluator/Self-Reflection 三角架构和语言强化学习范式
  [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)

- **Yao et al. (2022)** — ReAct 原论文，Reflexion 的 Actor 层基础
  [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)

- **Prompt Engineering Guide** — Reflexion 技术介绍，含 AlfWorld/HotPotQA/HumanEval 实验结果
  [https://www.promptingguide.ai/techniques/reflexion](https://www.promptingguide.ai/techniques/reflexion)

- **Eric Jang (2023)** — "Can LLMs Critique and Iterate on Their Own Outputs?" — 早期对 LLM 自我反思能力的实验性分析
  [https://evjang.com/2023/03/26/self-reflection.html](https://evjang.com/2023/03/26/self-reflection.html)

关联 wiki：[[wiki/query-loop]] · [[wiki/evaluation-observability]]
