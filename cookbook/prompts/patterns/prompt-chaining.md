---
pattern: prompt-chaining
category: orchestration
tags: [pipeline, decomposition, multi-step, workflow, agent-orchestration]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_wiki: [wiki/query-loop, wiki/multi-agent, wiki/prompt-system]
related_patterns: [chain-of-thought, react, structured-output]
---

# Prompt Chaining

> 将复杂任务拆成多个简单 prompt 的流水线，前一步的输出作为后一步的输入，逐步完成目标。

## 本质

一个人做不好的大事，拆成多件小事逐个搞定。

当你把一个复杂任务用一个超长 prompt 喂给 LLM，模型要同时处理太多东西：理解要求、拆解问题、做判断、输出格式……注意力分散，质量自然下降。

Prompt Chaining 的思路很朴素：把大任务切开，每步只做一件事。Step 1 只提取引用，Step 2 才用引用来回答。每步的 prompt 更短、意图更清晰、输出更可控。错了也知道在哪步出的问题。

与 ReAct 的根本区别：Prompt Chaining 的步骤序列是**设计时确定**的（你知道要走哪几步），ReAct 是**运行时动态决定**的（走一步看一步）。

## 什么时候用

- **单 prompt 质量明显下降**：任务太复杂，一口气做质量不达标，拆开每步都做得更好
- **需要中间验证**：某一步结果需要人工审查或程序校验，再决定是否继续
- **不同步骤需要不同角色**：Step 1 是"信息提取者"，Step 2 是"分析师"，Step 3 是"写作者"——每步 system prompt 不同
- **Agent pipeline 编排**：Orchestrator 把大任务拆成子任务派给 Workers，每个 Worker 的输出传给下一个
- **需要可审计性**：每步的输入输出都留存，方便回溯哪步出了问题

## 什么时候不用

- **简单任务**：一步就能做好的事，硬拆成多步只增加延迟和 token 消耗
- **步骤强依赖全局 context**：每步都需要看完整的原始信息，context 无法精简，拆了反而更贵
- **延迟敏感场景**：每步都要发一次 LLM 请求，多步串行延迟叠加，实时对话场景要谨慎
- **步骤不固定**：不知道需要几步、需要哪些步骤——这种情况用 ReAct，让模型自己决定

## 模板

### 基础版：线性 Chain

最常见形态。Step 1 的输出直接作为 Step 2 的输入变量。以文档问答为例：

**Step 1 Prompt — 提取引用**

```text
你是一个信息提取助手。你的任务是从文档中找出与问题相关的原文片段。

文档内容（被 #### 包围）：
####
{{document}}
####

问题：{{question}}

请从文档中提取所有与问题相关的原文句子或段落，用 <quotes></quotes> 标签包裹输出。
如果找不到相关内容，输出：<quotes>未找到相关内容</quotes>
```

**Step 1 输出示例：**

```xml
<quotes>
- Prompt chaining is useful to accomplish complex tasks which an LLM might struggle to address if prompted with a very detailed prompt.
- Prompt chaining helps to boost the transparency of your LLM application, increases controllability, and reliability.
</quotes>
```

**Step 2 Prompt — 基于引用作答**

```text
你是一个问答助手。根据以下从文档中提取的相关原文片段，回答用户的问题。
回答要准确、友好、有帮助。不要编造原文中没有的信息。

相关原文片段：
{{step1_output}}

问题：{{question}}
```

**Python 实现骨架：**

```python
import anthropic

client = anthropic.Anthropic()

def linear_chain(document: str, question: str) -> str:
    # Step 1: 提取相关引用
    step1_response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        messages=[{
            "role": "user",
            "content": STEP1_PROMPT.format(document=document, question=question)
        }]
    )
    quotes = step1_response.content[0].text

    # 可选：验证 Step 1 输出（质量门禁）
    if "未找到相关内容" in quotes:
        return "文档中没有找到与问题相关的内容。"

    # Step 2: 基于引用作答
    step2_response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": STEP2_PROMPT.format(step1_output=quotes, question=question)
        }]
    )
    return step2_response.content[0].text
```

---

### 基础版：Map-Reduce Chain

并行处理多个片段，再汇总结果。适合处理超长文档、批量数据分析。

```python
import asyncio
import anthropic

client = anthropic.Anthropic()

async def map_reduce_chain(chunks: list[str], task: str) -> str:
    """
    Map 阶段：并行处理每个 chunk
    Reduce 阶段：汇总所有结果
    """
    # Map：并行处理（每个 chunk 独立，无依赖）
    async def process_chunk(chunk: str, idx: int) -> str:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=512,
            messages=[{
                "role": "user",
                "content": f"任务：{task}\n\n文本片段：\n{chunk}\n\n请完成上述任务，只处理这段文本，简洁输出。"
            }]
        )
        return f"[片段 {idx + 1}]\n{response.content[0].text}"

    map_results = await asyncio.gather(*[
        process_chunk(chunk, i) for i, chunk in enumerate(chunks)
    ])

    # Reduce：汇总所有 Map 结果
    combined = "\n\n".join(map_results)
    reduce_response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        messages=[{
            "role": "user",
            "content": f"以下是对多个文本片段分别完成「{task}」的结果，请综合整理成一份连贯的最终输出：\n\n{combined}"
        }]
    )
    return reduce_response.content[0].text
```

---

### Agent 集成版：Orchestrator → Workers → 汇总

这是 Prompt Chaining 在多 agent 系统中的真实形态。Orchestrator 负责拆任务和汇总，Workers 各自执行一个子任务。

**Orchestrator System Prompt：**

```text
你是一个任务编排者。你的职责是：
1. 将用户的复杂请求分解为独立的子任务
2. 以结构化 JSON 格式输出任务列表
3. 收到所有子任务结果后，综合汇总成最终答案

分解规则：
- 每个子任务必须是独立的、可以单独完成的
- 子任务要足够小，一个 LLM 一步就能完成
- 如果子任务之间有顺序依赖，在 depends_on 字段中注明

输出格式（分解阶段）：
{
  "tasks": [
    {
      "id": "t1",
      "role": "researcher",  // 该子任务的执行角色
      "instruction": "...",  // 具体指令
      "depends_on": []       // 依赖的任务 id 列表
    }
  ]
}
```

**Worker System Prompt 模板：**

```text
你是一个{{role}}。你只负责完成被分配的单一子任务，不要超出范围。

子任务：{{instruction}}

{{#if context}}
上游任务结果（供参考）：
{{context}}
{{/if}}

直接输出任务结果，不需要解释你在做什么。
```

**编排循环骨架（Python）：**

```python
import json
import anthropic

client = anthropic.Anthropic()

def run_orchestrator_chain(user_request: str) -> str:
    # Phase 1: Orchestrator 拆任务
    decompose_response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=ORCHESTRATOR_SYSTEM_PROMPT,
        messages=[{"role": "user", "content": f"请分解以下任务：{user_request}"}]
    )
    task_plan = json.loads(decompose_response.content[0].text)
    tasks = task_plan["tasks"]

    # Phase 2: Workers 按依赖顺序执行
    results = {}
    for task in tasks:
        # 收集上游依赖结果
        upstream_context = "\n\n".join(
            f"[{dep}] {results[dep]}" for dep in task["depends_on"] if dep in results
        )

        worker_response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=2048,
            system=WORKER_SYSTEM_PROMPT.format(
                role=task["role"],
                instruction=task["instruction"],
                context=upstream_context
            ),
            messages=[{"role": "user", "content": "请完成你的任务。"}]
        )
        results[task["id"]] = worker_response.content[0].text

    # Phase 3: Orchestrator 汇总
    summary_input = "\n\n".join(
        f"[子任务 {tid}]\n{result}" for tid, result in results.items()
    )
    final_response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        system=ORCHESTRATOR_SYSTEM_PROMPT,
        messages=[{
            "role": "user",
            "content": f"原始任务：{user_request}\n\n各子任务结果：\n{summary_input}\n\n请综合汇总最终答案。"
        }]
    )
    return final_response.content[0].text
```

**错误处理策略：**

```python
def execute_with_retry(task: dict, context: str, max_retries: int = 2) -> str:
    """带重试和质量验证的 Worker 执行"""
    for attempt in range(max_retries + 1):
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=2048,
            system=WORKER_SYSTEM_PROMPT.format(**task, context=context),
            messages=[{"role": "user", "content": "请完成你的任务。"}]
        )
        result = response.content[0].text

        # 质量门禁：验证输出是否满足要求
        if validate_output(result, task):
            return result

        if attempt < max_retries:
            # 把失败原因反馈给模型，让它重试
            context += f"\n\n[注意] 上次输出不符合要求，请重新完成。"

    # 所有重试失败，返回失败标记，让 Orchestrator 降级处理
    return f"[FAILED] 任务 {task['id']} 无法完成"
```

---

### 变体速查

| 变体 | 结构 | 核心机制 | 适用场景 |
|------|------|----------|----------|
| **Linear** | A → B → C | 顺序传递，每步输出是下步输入 | 步骤有严格先后依赖，如提取→分析→撰写 |
| **Branching** | A → [B1 or B2] → C | 条件路由，Step 1 输出决定走哪条路 | 不同类型输入需要不同处理路径 |
| **Loop** | A → B → [验证] → A（如未通过） | 循环检查，直到满足质量标准 | 代码生成+测试、内容生成+审查 |
| **Map-Reduce** | [A1, A2, A3] → B | 并行 map + 串行 reduce | 长文档分块处理、批量数据汇总 |
| **Pipeline with Gates** | A → [Gate] → B → [Gate] → C | 每步后设质量门禁，不达标阻断 | 高风险输出（法律、医疗）、多级审查流程 |

**Branching 示例：**

```python
def branching_chain(user_input: str) -> str:
    # Step 1: 分类路由
    route_response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=64,
        messages=[{
            "role": "user",
            "content": f"判断以下输入属于哪类任务，只回答 'code'、'analysis' 或 'writing'：\n{user_input}"
        }]
    )
    task_type = route_response.content[0].text.strip().lower()

    # Step 2: 根据分类选择对应处理 prompt
    prompts = {
        "code": CODE_SPECIALIST_PROMPT,
        "analysis": ANALYSIS_SPECIALIST_PROMPT,
        "writing": WRITING_SPECIALIST_PROMPT,
    }
    specialist_prompt = prompts.get(task_type, DEFAULT_PROMPT)

    response = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        system=specialist_prompt,
        messages=[{"role": "user", "content": user_input}]
    )
    return response.content[0].text
```

## 组合与选择

**Prompt Chaining + CoT**
在 chain 的某一步里让模型先做 CoT 推理，输出结构化中间结论，再传给下一步。适合需要显式推理过程的子任务（如法律分析、数学推导）。

**Prompt Chaining + ReAct**
宏观用 Prompt Chaining 确定固定步骤序列，微观在某些步骤内嵌 ReAct loop 动态获取信息。适合流程固定但某步需要动态工具调用的场景（如：固定三步流程，但 Step 2 需要联网搜索）。

**Prompt Chaining + Structured Output**
每步输出用 JSON schema 约束，确保下一步能精确解析。在 pipeline 中间步骤强烈推荐，避免文本解析的脆弱性。

**Prompt Chaining vs ReAct**

| 维度 | Prompt Chaining | ReAct |
|------|-----------------|-------|
| 步骤结构 | 固定（设计时确定） | 动态（运行时决定） |
| 适用场景 | 流程确定的任务 | 路径不确定的任务 |
| 调试难度 | 低（每步输入输出明确） | 中（需要看 Thought 日志） |
| 延迟 | 可并行化（Map-Reduce） | 串行循环，延迟不可控 |
| 灵活性 | 低 | 高 |

结论：**预先知道需要哪些步骤 → Prompt Chaining；不知道需要几步 → ReAct**。

**Prompt Chaining vs 单次 CoT**

| 维度 | 单次 CoT | Prompt Chaining |
|------|----------|-----------------|
| 实现复杂度 | 低 | 中 |
| 可控性 | 低（黑盒推理） | 高（每步可验证） |
| 适用任务规模 | 中等复杂 | 高度复杂、多阶段 |
| Context 消耗 | 一次性 | 多次调用，但每次更小 |

## 模型差异

| 模型 | 指令遵循 | 结构化输出 | 长 chain 稳定性 | 注意事项 |
|------|----------|------------|-----------------|----------|
| **Claude 4** (Sonnet/Opus) | 稳定，边界清晰 | XML/JSON 均可靠 | 好，不易跑偏 | 建议用 XML 标签包裹中间输出（`<result>`），模型天然适配 |
| **GPT-4o** | 稳定 | JSON 强，XML 一般 | 好 | function calling 做结构化输出比 prompt 约束更稳 |
| **Gemini 2** | 良好 | JSON 稳 | 长 chain 后期偶有偏移 | 每步 prompt 保持自包含（不假设模型记住前步系统 prompt）|
| **开源模型** | 参数小时易跑偏 | 需要更严格的格式约束 | 差异大 | 建议每步都加 few-shot 示例；步骤越多越需要更大参数量的模型 |

**通用建议**：每步 prompt 都要自包含（包含当前步骤所需的全部信息），不要依赖模型"记住"前几步的指令。

## 常见踩坑

**1. 信息丢失**

症状：Step 1 产出了关键信息，但 Step 2 输出却漏掉了这部分内容。

原因：向 Step 2 传递的上下文被过度裁剪，或 Step 2 的 prompt 没有明确要求利用上游输出。

解决方案：
- Step 2 prompt 明确写："你必须基于以下上游结果作答，不能无视它"
- 保留上游输出的原始格式（不要手动裁剪后传入）
- 在传递前用 XML/JSON 结构化包裹，让模型更容易"定位"到关键信息

**2. 错误传播**

症状：Step 1 输出了错误信息，后续所有步骤都在此基础上继续错下去，最终结果全错。

原因：没有在步骤之间设置质量门禁，错误无法被阻断。

解决方案：
- 关键步骤后加验证逻辑（程序校验或让另一个 LLM 审查）
- 对高风险输出设置 Pipeline with Gates 模式
- Step N+1 的 prompt 可以加："如果上游结果存在明显错误或缺失，请先指出，再继续"

**3. 过度拆分**

症状：任务被拆成了 8 步，但其中 5 步完全可以合并，chain 越来越长却没有带来质量提升。

原因：为了"拆"而拆，没有评估每个拆分点是否真的必要。

解决方案：
- 每个拆分点必须有理由：中间需要验证、需要不同角色、或单步质量确实更好
- 先用单 prompt 做一次，看哪个子任务质量差，再针对性拆开
- 两步合并原则：如果 Step A 的输出在 Step B 里几乎原封不动地被用到，可以合并

**4. Context 重复导致 Token 爆炸**

症状：每步都把原始文档全量传入，3 步下来 token 消耗是单步的 3 倍，而且越来越贵。

原因：没有在步骤间做 context 精简，每步 prompt 都携带全量信息。

解决方案：
- Step 1 的职责就是提炼关键信息（如提取引用），后续步骤只传 Step 1 的输出，不再传原始文档
- Map 阶段做信息压缩，Reduce 阶段处理压缩后的摘要，而不是全量 chunk
- 对超长中间结果，用 Structured Output 约束输出格式，避免模型写废话撑大 context

**5. 每步 Prompt 相互依赖隐式 System Prompt**

症状：Step 2 的 prompt 假设模型"知道"Step 1 里的背景设定，但实际上每次 API 调用是独立的。

原因：忘记了每步都是一次全新的 LLM 调用，没有上下文继承。

解决方案：
- 每步 prompt 必须自包含，把必要背景信息显式写进去
- 如果背景信息很长，用 system prompt 传递，让 prompt caching 降低成本
- 在框架层面封装 context 传递逻辑，而不是每次手动拼接

## 来源

- **Prompt Engineering Guide** — Prompt Chaining 技术概述与文档 QA 示例
  [https://www.promptingguide.ai/techniques/prompt_chaining](https://www.promptingguide.ai/techniques/prompt_chaining)

- **Anthropic Claude Docs** — Prompt Chaining 官方指南，附 Claude 专项示例
  [https://docs.anthropic.com/claude/docs/prompt-chaining](https://docs.anthropic.com/claude/docs/prompt-chaining)

- **DeerFlow** — 多 agent pipeline 中 Orchestrator-Workers 模式的工程实现参考
  `raw/deer-flow`

关联 wiki：[[wiki/query-loop]] · [[wiki/multi-agent]] · [[wiki/prompt-system]]
