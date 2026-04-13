---
pattern: react
category: action
tags: [agent-loop, tool-use, reasoning-acting, multi-step, agent-core]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_wiki: [wiki/query-loop, wiki/tool-system, wiki/prompt-system]
related_patterns: [chain-of-thought, structured-output, prompt-chaining]
---

# ReAct

> 让模型在推理（Thought）和行动（Action）之间交替循环，根据观察结果动态调整下一步，是大多数 Agent 的核心运行模式。

## 本质

人类解决问题的方式从来不是"先想清楚所有步骤，再一口气做完"。我们会想一下，试一下，看看结果，再想，再试。遇到意外就调整，发现路走错了就回头。

ReAct 把这个人类自然的问题解决过程形式化给了 LLM：

- **Thought**：当前怎么理解这个问题？下一步该做什么？为什么？
- **Action**：调用工具（搜索、计算、查数据库、执行代码……）
- **Observation**：工具返回了什么？
- 回到 Thought，基于新信息继续推进

这个循环持续到任务完成。每次 Observation 都是真实世界的反馈，让模型的推理始终扎根于事实，而不是凭空想象。

与纯 CoT 的本质区别：CoT 只是在 LLM 的"脑子里"推理，ReAct 能真正出去取信息、改变外部状态。

## 什么时候用

- **需要调用外部工具获取信息**：查当前天气、搜索最新数据、查数据库、调 API——这些信息在模型训练集里没有或已过时
- **多步任务且每步依赖上步结果**：不知道要搜什么，直到看了第一步的结果；不知道要算什么，直到查到了某个数值
- **Agent 主循环**：几乎所有 agent 框架（LangChain、LlamaIndex、neoagent 等）的核心都是 ReAct 的变体，这是 agent 的默认运行模式
- **需要可解释性**：Thought 步骤完整暴露了模型的决策过程，方便调试和审查
- **任务路径不确定**：预先不知道需要几步、需要哪些工具，只能走一步看一步

## 什么时候不用

- **纯推理，不需要外部信息**：数学题、代码逻辑分析、文本改写等——直接用 CoT，引入 ReAct 循环只增加延迟和 token 消耗
- **单次 tool call 就能搞定**：只需查一次价格、翻译一段文字——不需要循环，直接 function calling 拿结果
- **批量同质任务**：100 个文件格式转换、1000 条数据标注——直接 map，不需要动态决策
- **极低延迟要求**：每轮 ReAct 循环都需要 LLM 推理 + 工具调用，延迟叠加明显；实时对话场景要谨慎

## 模板

### 基础版

学术原版格式，适合理解概念和快速原型。通过 few-shot 示例让模型学会格式，然后在问答中触发 Thought/Action/Observation 循环。

```text
你是一个能使用工具回答问题的助手。按照以下格式思考和行动：

Thought: [分析当前情况，决定下一步]
Action: [工具名称]([参数])
Observation: [工具返回的结果]
... (重复 Thought/Action/Observation 直到有答案)
Thought: 我现在知道最终答案了
Final Answer: [最终回答]

可用工具：
- Search(query): 搜索网络信息
- Calculate(expression): 计算数学表达式
- Lookup(term): 在当前文章中查找特定术语

---

问题：苹果公司最新一季度的营收是多少？与上一季度相比变化了多少？

Thought: 我需要找苹果公司最新季度的营收数据，然后再找上一季度的数据进行对比。先搜索最新季度。
Action: Search(Apple Inc latest quarterly revenue 2024)
Observation: 苹果公司 2024 财年 Q4（截至 2024 年 9 月）营收为 949 亿美元，同比增长 6%。

Thought: 我有了最新季度的数据。现在需要上一季度（Q3）的数据来计算环比变化。
Action: Search(Apple Inc Q3 2024 revenue)
Observation: 苹果公司 2024 财年 Q3（截至 2024 年 6 月）营收为 857 亿美元。

Thought: 我现在有了两个季度的数据：Q4 为 949 亿美元，Q3 为 857 亿美元。
计算环比增长：(949 - 857) / 857 × 100% ≈ 10.7%
Action: Calculate((949 - 857) / 857 * 100)
Observation: 10.734...

Thought: 我现在知道最终答案了。
Final Answer: 苹果公司 2024 财年 Q4 营收为 949 亿美元，Q3 为 857 亿美元，环比增长约 10.7%。
```

### Agent 集成版

这是 ReAct 在生产环境中的真实形态。现代 agent 框架不再用文本模式匹配解析 Thought/Action，而是用 function calling API 来结构化地处理 Action 步骤。Thought 依然存在，通常作为模型在调用工具前的文字推理，或嵌入在 tool_use 的参数里。

**核心变化：**
- Action 变成了 `tool_use` API 调用（JSON 结构化，不用文本解析）
- Observation 变成了 `tool_result` 消息
- Thought 变成了模型在 `text` block 里的自然语言推理
- 循环由 agent 框架的外层 while loop 驱动，而不是 prompt 里的 few-shot 示例

**System Prompt 模板（生产级）：**

```text
你是一个任务执行助手。你可以使用以下工具来完成用户的请求。

## 可用工具

{tools_list}
（由框架动态注入，例如：web_search、calculator、code_executor、file_reader）

## 工作方式

1. 分析用户的请求，判断是否需要工具
2. 如果需要工具，调用最合适的工具
3. 查看工具返回结果，决定下一步
4. 重复直到任务完成，然后直接回答用户

## 退出条件（何时停止循环）

满足以下任一条件时，不再调用工具，直接给用户最终答案：
- 已经收集到足够的信息来回答问题
- 任务已经完成（文件已写入、操作已执行等）
- 工具连续返回错误且无法绕过
- 已达到最大轮次限制（由框架设置，通常 10-20 轮）

## 重要原则

- 每次只调用一个工具（除非框架支持并行 tool call）
- 工具失败时，先尝试换参数重试，再考虑换工具
- 不要重复调用刚才已经失败的工具用完全相同的参数
- 如果你已经知道答案，不要为了"确认"而多余地调用工具
```

**Tool Definition 示例（Anthropic Claude API）：**

```python
tools = [
    {
        "name": "web_search",
        "description": "搜索互联网获取最新信息。适用于需要实时数据、新闻、或训练集截止日期后的信息。",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "搜索查询词，使用英文效果更好"
                },
                "num_results": {
                    "type": "integer",
                    "description": "返回结果数量，默认 5，最多 10",
                    "default": 5
                }
            },
            "required": ["query"]
        }
    },
    {
        "name": "calculator",
        "description": "执行数学计算。支持基本运算、三角函数、对数等。",
        "input_schema": {
            "type": "object",
            "properties": {
                "expression": {
                    "type": "string",
                    "description": "数学表达式，例如 '(949 - 857) / 857 * 100'"
                }
            },
            "required": ["expression"]
        }
    }
]
```

**Agent 循环骨架（Python）：**

```python
import anthropic

client = anthropic.Anthropic()

def run_react_agent(user_message: str, max_turns: int = 20) -> str:
    messages = [{"role": "user", "content": user_message}]
    
    for turn in range(max_turns):
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=4096,
            system=SYSTEM_PROMPT,  # 上方的 system prompt 模板
            tools=tools,
            messages=messages
        )
        
        # 退出条件 1：模型决定直接回答（stop_reason = "end_turn"）
        if response.stop_reason == "end_turn":
            # 提取最后的文字回答
            return next(
                block.text for block in response.content 
                if hasattr(block, "text")
            )
        
        # 退出条件 2：需要调用工具（stop_reason = "tool_use"）
        if response.stop_reason == "tool_use":
            # 把模型的回复加入消息历史
            messages.append({"role": "assistant", "content": response.content})
            
            # 执行所有工具调用，收集结果
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)  # 你的工具执行逻辑
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })
            
            # 把工具结果加入消息历史，触发下一轮推理
            messages.append({"role": "user", "content": tool_results})
            continue
    
    # 退出条件 3：达到最大轮次
    return "达到最大轮次限制，任务未完成。"
```

**关键设计点：**

1. **退出条件设计**：何时停止循环直接影响 agent 的行为质量。退出太早 → 任务做一半；退出太晚 → 浪费 token、可能绕圈。生产环境建议同时设置：模型主动停止（`end_turn`）+ 最大轮次兜底（`max_turns`）+ 工具连续失败熔断。

2. **并行 tool call**：Claude 和 GPT-4 都支持在一次回复里调用多个工具。当多个工具相互独立时（例如同时搜索 A 和 B），启用并行可显著降低延迟。

3. **消息历史管理**：每轮循环后，完整的 Thought + tool_use + tool_result 都留在 messages 里。这是"工作记忆"，但也意味着长任务会撑爆 context window。需要配合压缩策略。

### 变体

| 变体 | 格式 | 核心机制 | 适用场景 | 主要取舍 |
|------|------|----------|----------|----------|
| **Classic ReAct** | 文本格式（Thought/Action/Observation） | Few-shot 示例驱动，正则/文本解析 Action | 教学、快速原型、不支持 function calling 的模型 | 简单易懂，但解析不稳定、格式容易乱 |
| **Function-calling ReAct** | tool_use API | 结构化 JSON，框架驱动循环 | 现代生产环境的标准实现 | 稳定可靠，但需要模型支持 tool_use |
| **Reflexion** | ReAct + 事后反思层 | 失败后生成 self-reflection 存入记忆，重试 | 需要多次尝试才能成功的任务（编程、规划） | 效果好，但每次失败都多一轮 LLM 调用 |
| **PlanAct** | 先规划再执行 | 第一步生成完整计划，后续步骤按计划执行 | 任务结构明确、步骤可预知 | 减少中途决策，但规划失误全盘皆输 |

**选择建议：**
- 默认用 Function-calling ReAct（稳定、现代）
- 任务失败率高 → 加 Reflexion 层
- 任务结构清晰 → 考虑 PlanAct（可以减少 context 消耗）
- 只有文本 API → Classic ReAct

## 组合与选择

**ReAct + CoT**
Thought 步骤本身就是 CoT。每次循环里，模型在 Thought 里做链式推理，推导出下一个 Action。两者天然融合，不是"二选一"的关系。

**ReAct + Structured Output**
现代实现里，Action 步骤就是 Structured Output 的 JSON schema（function calling）。用 JSON schema 约束工具调用参数，避免参数格式错误导致工具执行失败。

**ReAct + Prompt Chaining**
宏观用 Prompt Chaining 分解大任务（固定步骤序列），微观在每个步骤内用 ReAct 处理动态信息获取。适合流程固定但每步需要联网的场景。

**ReAct vs 纯 CoT**

| 维度 | 纯 CoT | ReAct |
|------|--------|-------|
| 信息来源 | 仅模型训练集 | 训练集 + 实时工具 |
| 适用场景 | 推理、写作、分析 | 需要外部信息的任务 |
| 延迟 | 单次推理，快 | 多轮循环，慢 |
| 幻觉风险 | 较高（无事实校验） | 较低（工具提供 ground truth） |

结论：**有工具就用 ReAct，没工具就用 CoT**。

**ReAct vs Prompt Chaining**

| 维度 | Prompt Chaining | ReAct |
|------|-----------------|-------|
| 步骤结构 | 固定（设计时确定） | 动态（运行时决定） |
| 适用场景 | 流程确定的任务 | 路径不确定的任务 |
| 调试难度 | 低（步骤可见） | 中（需要看 Thought 日志） |
| 灵活性 | 低 | 高 |

结论：**预先知道需要哪些步骤 → Prompt Chaining；不知道需要几步 → ReAct**。

## 模型差异

| 模型 | tool_use 支持 | Thought 质量 | 并行 tool call | 注意事项 |
|------|--------------|-------------|----------------|----------|
| **Claude 4** (Sonnet/Opus) | 原生支持，稳定 | 推理链完整，较少跳步 | 支持 | 退出条件清晰时很少死循环；Thought 文字偏长 |
| **GPT-4o** | 原生支持，稳定 | 推理质量高 | 支持（parallel_tool_calls） | 有时工具调用后直接输出结果，不补充 Thought |
| **Gemini 2** | 支持（function calling） | 在复杂多步任务上偶有偏差 | 支持 | 长对话历史下上下文追踪稳定性略差 |
| **开源模型**（Llama 3, Qwen 2.5 等） | 依赖微调质量，差异大 | 参数小的模型容易格式化失败 | 部分支持 | 建议用 Classic ReAct 文本格式，更鲁棒；Hermes-3 等工具调用微调模型表现较好 |

**工程建议**：无论哪个模型，都要设置 `max_turns` 兜底。开源模型额外建议做 Thought 内容的有效性验证（空 Thought 或重复 Thought 是死循环信号）。

## 常见踩坑

**1. 无限循环**

症状：模型一直调用工具，永远不输出最终答案。

原因：退出条件不清晰，或模型对"什么时候算完成"没有明确指引。

解决方案：
- System prompt 里显式写退出条件（见 Agent 集成版模板）
- 代码层面强制 `max_turns` 限制
- 监控重复 Action（连续 3 次相同工具调用相同参数 → 强制停止）

**2. 过度思考**

症状：Thought 越来越长，模型在 Thought 里把问题分析了又分析，但一直不行动。

原因：System prompt 没有强调"分析够了就行动"，或模型对工具调用的"授权"不够明确。

解决方案：
- Prompt 里加："当你有足够信息调用工具时，立即行动，不要在 Thought 里过度分析"
- 限制 Thought 长度（提示词层面）
- 对于明确的工具调用场景，用更指令性的 prompt 风格

**3. 工具选择错误**

症状：模型选了不适合当前子任务的工具，导致 Observation 无效或错误，后续推理跑偏。

原因：工具 description 写得不够清晰，或工具之间的边界模糊。

解决方案：
- 工具 description 要写清楚"适用于"和"不适用于"
- 在 description 里给出 1-2 个典型用例
- 工具数量多时，考虑分组或用 router 先选工具类别

**4. 观察结果被忽略**

症状：工具返回了有效信息，但模型在下一个 Thought 里没有利用这个信息，依然按原计划行动。

原因：Context window 里的 tool_result 被后续 token 稀释，或模型对 observation 的重视程度不足。

解决方案：
- Prompt 里加："在每次 Thought 开始时，先总结最近一次 Observation 的关键信息"
- 重要 Observation 可以在 tool_result 前加前缀标记（如 `[重要发现]`）
- 注意 context 长度：过长的历史会让早期 Observation 被"遗忘"，适时压缩

## 来源

- **Yao et al. (2022)** — ReAct 原论文，提出 Thought/Action/Observation 三步范式
  [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)

- **Shinn et al. (2023)** — Reflexion 论文，在 ReAct 基础上加入自我反思和记忆
  [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366)

- **Prompt Engineering Guide** — ReAct prompting 示例和 HotpotQA 演示
  [https://www.promptingguide.ai/techniques/react](https://www.promptingguide.ai/techniques/react)

- **Anthropic Claude API Docs** — tool_use API 和 function calling 规范
  [https://docs.anthropic.com/en/docs/build-with-claude/tool-use](https://docs.anthropic.com/en/docs/build-with-claude/tool-use)

关联 wiki：[[wiki/query-loop]] · [[wiki/tool-system]] · [[wiki/prompt-system]]
