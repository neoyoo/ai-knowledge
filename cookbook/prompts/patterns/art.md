---
pattern: art
category: action
tags: [automatic-reasoning, tool-selection, few-shot-tools, task-library]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: advanced
related_wiki: [wiki/tool-system, wiki/query-loop]
related_patterns: [react, chain-of-thought, pal]
---

# ART (Automatic Reasoning and Tool-use)

> 自动从任务库中检索相关示例，生成包含推理步骤和工具调用的解决方案，减少手动 prompt 工程。

## 本质

想象你是一个刚入职的工程师，桌上有一本前人积累的《解题手册》。来了新问题，先翻手册找类似案例，照着案例的思路（推理步骤）和工具选择来解决——而不是每次都从零开始想。ART 对 LLM 做的就是这件事：给模型一个"参考手册"（任务库），遇到新任务时自动检索最相似的历史解法（包括推理链和工具调用顺序），然后仿照执行。

核心机制分两步：

1. **检索阶段**：给定新任务，从任务库中选取语义最相近的 multi-step 示例作为 few-shot context
2. **暂停-恢复循环**：生成过程中遇到工具调用标记时，暂停 LLM 生成 → 执行工具 → 把结果插回上下文 → 恢复生成

和 ReAct 的核心区别：ReAct 的"应该用什么工具、怎么用"是每次由模型实时决策；ART 的工具选择模式来自示例库的泛化，更像是"对照参考案例模仿执行"。ART 是 ReAct 的前身之一，发表于 2023 年，当时 function calling API 尚未普及，现代 tool use API 已经部分取代了它的文本格式工具调用机制，但其任务库检索思路在 RAG-augmented agent 设计中仍有参考价值。

## 什么时候用

**有大量同类任务、想减少逐个写 prompt 的工作量** — 比如数据分析流水线（每次都要：加载 → 清洗 → 统计 → 可视化），把标准流程存入任务库，新任务自动复用。

**任务需要混合推理和工具调用** — 纯推理用 CoT，纯工具调用用 function calling；当一个任务既需要多步推理又需要在推理中途调用工具时，ART 提供了结构化的组合方式。

**工具选择规律可以从历史中学习** — 如果你的系统已经有一批"这类问题应该用这几个工具"的历史案例，ART 的任务库机制可以把这些知识编码进去，让模型自动泛化。

**构建领域专用 agent 原型** — 在没有完整 function calling 生态、或者需要对工具调用序列有细粒度控制的场景下，ART 的文本格式工具调用提供了一种灵活的原型方案。

## 什么时候不用

**任务类型单一** — 如果所有任务都是同一类，直接写一个精确的 system prompt 效果更好、更可控，不需要检索机制。

**没有任务库或历史案例** — ART 的价值来自任务库，没有库就退化成普通 few-shot。冷启动时需要先积累足够的示例。

**需要精确控制每一步** — 任务库示例的泛化是概率性的，模型可能选错示例或错误模仿。对每步都需要确定性保证的场景，用显式工具调度逻辑（如 orchestrator 代码）更可靠。

**现代 function calling 已经够用** — GPT-4、Claude 3+、Gemini 2 的 native function calling 在结构化工具调用方面已经超过 ART 的文本格式方案。只有当你需要"自动示例检索"这个额外能力时，才值得引入 ART 的任务库机制。

## 模板

### 基础版：任务库 few-shot 结构

ART 的 prompt 结构是在标准 few-shot 基础上加入工具调用标记：

```text
完成以下任务。遇到需要工具的步骤时，使用 [TOOL: 工具名(参数)] 格式调用，
等待工具返回结果后继续推理。

# 任务示例 1
任务：计算 2020 年到 2023 年全球 EV 销量的年均增长率
推理：
  步骤1：需要获取各年销量数据
  [TOOL: search("全球EV销量 2020 2021 2022 2023")]
  结果：2020: 310万辆, 2021: 650万辆, 2022: 1020万辆, 2023: 1400万辆
  步骤2：计算 CAGR = (终值/初值)^(1/n) - 1
  CAGR = (1400/310)^(1/3) - 1 ≈ 65.6%
答案：年均复合增长率约为 65.6%

# 任务示例 2
任务：查找 Python requests 库的最新版本并检查是否有安全漏洞
推理：
  步骤1：查询最新版本
  [TOOL: search("requests python library latest version")]
  结果：requests 2.31.0（2023年5月）
  步骤2：检查 CVE 数据库
  [TOOL: search("CVE requests python 2.31.0")]
  结果：无已知严重漏洞
答案：最新版本 2.31.0，无已知严重安全漏洞

# 新任务
任务：{{new_task}}
推理：
```

关键设计点：
- 示例中的推理步骤要涵盖你任务库中最常见的几种模式（数值计算、信息检索、代码分析等）
- `[TOOL: ...]` 标记需要外层代码解析并实际执行，结果回填后再继续生成
- 示例数量建议 2-4 个，覆盖不同工具组合，避免模型过拟合到单一模式

---

### Agent 集成版：任务路由 + 自动示例检索

在真实 agent 系统中，任务库检索通常由代码层实现，而不是把所有示例塞进 prompt：

```python
# 任务库存储（简化示意）
TASK_LIBRARY = [
    {
        "id": "math_with_lookup",
        "description": "需要先查询数据再进行数值计算的任务",
        "embedding": [...],  # 预计算的语义向量
        "demonstration": """
任务：计算某指标的同比增长率
推理：
  步骤1：检索当期和同期数据
  [TOOL: search("{query}")]
  结果：{tool_result}
  步骤2：计算增长率 = (当期 - 同期) / 同期 * 100%
答案：{final_answer}
"""
    },
    {
        "id": "code_debug",
        "description": "分析代码错误并查找修复方案的任务",
        "embedding": [...],
        "demonstration": """
任务：调试代码中的特定错误
推理：
  步骤1：分析错误类型和堆栈
  步骤2：搜索已知解决方案
  [TOOL: search("{error_type} fix")]
  结果：{tool_result}
  步骤3：验证修复方案适用性
答案：{solution}
"""
    }
]

def build_art_prompt(new_task: str, task_library: list, top_k: int = 2) -> str:
    # 1. 向量检索最相似示例
    task_embedding = embed(new_task)
    similarities = [cosine_sim(task_embedding, t["embedding"]) for t in task_library]
    top_indices = sorted(range(len(similarities)), key=lambda i: similarities[i], reverse=True)[:top_k]

    # 2. 组装 few-shot prompt
    demonstrations = "\n\n".join([task_library[i]["demonstration"] for i in top_indices])

    return f"""
完成以下任务。遇到需要工具的步骤时，使用 [TOOL: 工具名(参数)] 格式调用。

# 参考示例
{demonstrations}

# 新任务
任务：{new_task}
推理：
"""

def run_art_agent(task: str):
    prompt = build_art_prompt(task, TASK_LIBRARY)

    # 3. 暂停-恢复生成循环
    while True:
        response = llm.generate(prompt, stop=["[TOOL:"])

        if "[TOOL:" not in response:
            # 推理完成，返回最终答案
            return response

        # 解析工具调用
        tool_name, tool_args = parse_tool_call(response)

        # 执行工具并回填结果
        tool_result = execute_tool(tool_name, tool_args)
        prompt = prompt + response + f"\n  结果：{tool_result}\n"
```

这个模式的核心：**任务库检索在代码层做，LLM 只负责按示例格式生成推理链和工具调用标记**。这样任务库可以独立维护和扩展，不需要修改 prompt 模板。

---

### 与现代 function calling 的结合

如果你的场景已经有 native tool use API（Claude tool_use、OpenAI function calling），可以用 ART 的任务库思路来增强 system prompt，而不是完全替代 API：

```python
SYSTEM_PROMPT_TEMPLATE = """
你是一个数据分析助手，可以使用以下工具：{tool_descriptions}

## 参考解题模式

以下是处理类似任务的典型模式，参考这些模式来分解新任务：

{retrieved_demonstrations}

注意：参考模式是引导，不是约束。根据实际任务灵活调整工具调用顺序。
"""

def build_system_prompt(task: str) -> str:
    # 从任务库检索相关示例（只检索模式描述，不检索完整示例）
    relevant_patterns = retrieve_patterns(task, top_k=2)
    return SYSTEM_PROMPT_TEMPLATE.format(
        tool_descriptions=format_tools(),
        retrieved_demonstrations=format_patterns(relevant_patterns)
    )
```

这是 ART 思路在现代 API 下的务实变体：检索机制保留，工具调用格式交给 API 处理。

## 历史定位

ART 由 Paranjape et al. (2023) 提出，发表时间在 GPT-4 发布前后，彼时 function calling 尚未成为行业标准。它解决的核心问题是：如何让模型在没有精心手工设计的 prompt 的情况下，自动学会"什么时候用什么工具"。

**ART 在 agent 演化链中的位置：**

```
CoT (2022)                    → 推理外显化
ReAct (2023)                  → 推理 + 工具交织
ART (2023)                    → 推理 + 工具 + 自动示例检索
↓
Modern Function Calling API   → 结构化工具调用（取代文本格式 tool use）
↓
RAG-augmented Agent (现在)    → ART 的任务库思路在现代架构中的延续
```

ART 的文本格式工具调用（`[TOOL: ...]`）已被 native API 取代，但它的**任务库检索机制**——"遇到新任务，先找最相似的历史解法作为参考"——在现代 RAG 增强的 agent 系统中仍然有效，只是实现方式更加工程化。

## 模型差异

| 模型 | 表现描述 | 注意事项 |
|------|---------|---------|
| Claude 4 | 能很好地理解任务库示例的模式并泛化；native tool use 与 ART 任务库思路结合效果好 | extended thinking 开启时会对示例模式做更深分析；建议在 system prompt 中明确说明示例是"参考模式"而非"严格约束" |
| GPT-4 | 文本格式工具调用遵循率高；function calling 已经足够强大，ART 的附加价值主要在任务库检索这一层 | GPT-4o 的 function calling 在多工具选择上已超过 ART 的文本格式方案；用 ART 主要是为了复用示例库 |
| Gemini 2 | 对 few-shot 示例的模式泛化能力稳定；支持 native function calling | Gemini 2 Flash 在长示例上下文中偶有注意力漂移，建议示例不超过 3 个 |
| 开源模型（LLaMA 3、Qwen 2.5） | 70B+ 模型能较好遵循工具调用格式；任务库示例能有效引导工具选择 | 小于 13B 的模型对复杂工具调用格式遵循率低，建议简化工具标记格式；Qwen 2.5 在代码相关任务上的工具选择泛化效果较好 |

## 常见踩坑

**任务库需要持续人工维护** — 现象：任务库中的示例过时或不够覆盖，导致检索到的示例与新任务相关性低，模型用错误模式解决问题。原因：任务库是静态的，任务类型会随业务演化。避免方式：建立任务库的版本管理机制；定期审查低质量示例；在 agent 执行日志中记录检索命中率，作为任务库质量指标。

**示例检索质量直接影响结果** — 现象：检索到的示例工具调用模式与新任务不匹配，模型错误模仿，使用了不该用的工具或跳过了必要工具。原因：语义相似不等于解法相似，embedding 检索无法保证工具使用模式的匹配。避免方式：在任务库中加入"工具标签"字段，检索时同时考虑语义相似度和工具集合重叠度；或对任务做显式分类后再检索。

**暂停-恢复循环实现复杂** — 现象：工具调用解析错误、结果回填格式错乱，导致模型后续推理基于错误的工具结果。原因：ART 的文本格式工具调用需要自己实现解析器，边界情况多。避免方式：如果只是想要"工具调用嵌入推理链"的效果，优先考虑 ReAct 模式或直接用 native function calling；只在需要任务库检索这一层时才完整实现 ART。

**已被现代 tool use API 部分取代** — 现象：花大力气实现 ART 的文本格式工具调用，效果不如直接用 Claude/GPT-4 的 native function calling。原因：现代 API 在工具调用的结构化和可靠性上已经远超 2023 年的文本格式方案。避免方式：在现代系统中，只保留 ART 的"任务库检索"机制，工具调用本身交给 API；把 ART 理解为一种架构思路，而不是一套要完整复现的格式规范。

**过度依赖示例导致创造力缺失** — 现象：任务库示例覆盖不到的新任务类型，模型强行套用最相似示例的解法，得出错误答案。原因：few-shot 的过拟合效应被放大了——任务库让模型更倾向于"找前例"而不是"从零推理"。避免方式：在 prompt 中明确说明"以上是参考模式，如果新任务与任何示例都不匹配，请用你自己的判断分解任务"；或者在检索相似度低于阈值时，回退到 zero-shot CoT 模式。

## 来源

- Paranjape, B. et al. (2023). *ART: Automatic multi-step reasoning and tool-use for large language models*. arXiv. https://arxiv.org/abs/2303.09014
- Yao, S. et al. (2023). *ReAct: Synergizing Reasoning and Acting in Language Models*. ICLR 2023. https://arxiv.org/abs/2210.03629
- Wei, J. et al. (2022). *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. NeurIPS 2022. https://arxiv.org/abs/2201.11903
- Prompt Engineering Guide (DAIR.AI). ART: Automatic Reasoning and Tool-use. https://www.promptingguide.ai/techniques/art
- 关联 wiki: [[wiki/tool-system]] [[wiki/query-loop]]
