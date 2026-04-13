---
pattern: multimodal-cot
category: reasoning
tags: [multimodal, vision, image-reasoning, cross-modal]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_wiki: [wiki/prompt-system]
related_patterns: [chain-of-thought, structured-output]
---

# Multimodal CoT

> 将 CoT 推理扩展到多模态输入（文本 + 图像），让模型基于视觉信息进行分步推理，而不只是直接看图答题。

## 本质

普通"看图回答"是一步跳：图像 → 答案。Multimodal CoT 是两步走：图像+文本 → 推理过程 → 答案。

区别在哪里？第一步"生成推理"时，模型被迫先把图像中的关键视觉信息用语言表达出来，这个语言化的推理过程成为第二步"得出答案"的高质量输入。视觉感知和语言推理解耦，各自做好自己的事。

Zhang et al. (2023) 在 ScienceQA 基准上验证：1B 参数的 Multimodal CoT 模型超越了 GPT-3.5，因为 GPT-3.5 没有视觉输入。现代多模态大模型（Claude 4、GPT-4V、Gemini 2）原生支持图文混合输入，应用 Multimodal CoT 模式不需要额外微调，只需要正确构造 prompt。

**与普通 CoT 的区别**：普通 CoT 推理来源是文本上下文；Multimodal CoT 的推理来源包含图像——模型需要先"读懂图"，再"基于图推理"。

## 什么时候用

**科学题目/图表分析** — 解读实验图、数据图表、示意图时，先让模型描述图中的关键数据点，再基于数据推理结论。

**视觉问答（VQA）需要推理的场景** — 不是"图里有什么猫"这种感知问题，而是"图中的电路是否会短路"这种需要推理的问题。

**多图对比** — 对比两张图的异同、分析前后变化、评估设计方案差异。显式推理步骤让对比更系统。

**文档/截图理解** — 分析 UI 截图找问题、理解技术图表、解读医学影像（辅助场景）。先推理再结论的方式减少误判。

**Agent 处理视觉输入** — agent 收到图像后需要决策（下一步用哪个工具、当前状态是什么），CoT 让决策链路可见可调试。

## 什么时候不用

**纯感知任务** — "这张图里有几只猫"不需要推理，直接回答即可，加 CoT 是浪费 token。

**实时流式场景** — Multimodal CoT 的推理步骤让输出变长，对延迟要求高的场景权衡成本。

**图像质量很差时** — 如果图像本身模糊、分辨率过低，推理步骤也无法弥补感知层的缺陷，结果依然不可靠。

**没有视觉输入时** — 显然，没有图像就不需要这个 pattern，退化为普通 CoT。

## 模板

### 基础版：两阶段 prompt

```text
[阶段 1 - 视觉信息提取]
请仔细观察这张图像，描述以下内容：
1. 图像中的主要元素和它们的位置关系
2. 任何数字、标签、标注信息
3. 你认为与问题相关的关键视觉细节

[阶段 2 - 基于视觉推理]
基于你上面描述的视觉信息，现在回答：
{{question}}

请展示你的推理步骤，最后给出明确答案。
```

---

### 单次调用版（适合现代多模态模型）

```text
这是一道需要图像分析的问题。请按以下步骤回答：

步骤 1：描述图像中与问题相关的关键信息
步骤 2：基于这些信息进行推理
步骤 3：给出最终答案

图像：[附加图像]

问题：{{question}}
```

---

### 科学题目版

```text
请分析以下图像并回答问题。

图像：[附加图像]

分析过程：
- 图中显示的数据/现象：[请先描述你看到的]
- 根据这些信息，相关原理是：[推理过程]
- 因此，答案是：[结论]

问题：{{question}}
```

---

### Agent 集成版

```python
def build_multimodal_cot_prompt(question: str, image_context: str = None) -> list:
    """构建 Multimodal CoT 的消息列表"""
    system = (
        "你是一个视觉推理助手。"
        "收到图像时，先用语言描述图像的关键信息，再基于描述推理，最后给出答案。"
        "格式：[视觉观察] → [推理过程] → [最终答案]"
    )
    user_content = []
    if image_context:
        # 实际场景中这里放 base64 或 URL
        user_content.append({"type": "image", "source": image_context})
    user_content.append({"type": "text", "text": question})
    return [
        {"role": "system", "content": system},
        {"role": "user", "content": user_content},
    ]
```

## 组合与选择

**Multimodal CoT + Structured Output** → 推理过程自由展开，最终答案用固定格式输出（JSON、选项字母等）。适合需要机器解析答案的场景（如自动评测）。

**Multimodal CoT + Few-shot** → 提供 1-2 个"图像 + 推理过程 + 答案"的示例，教模型你期望的推理粒度。当 zero-shot Multimodal CoT 推理步骤过粗时使用。

**Multimodal CoT vs 直接描述图像** — 直接问"描述这张图"是感知任务；Multimodal CoT 是"基于图像推理问题答案"，目标不同。不要混用。

**单次调用 vs 两次调用** — 现代模型（Claude 4、GPT-4V）可以在单次调用里完成"感知→推理"；如果需要更细粒度的控制或中间结果验证，可以拆成两次调用（第一次提取视觉信息，第二次基于提取结果推理）。

## 模型差异

| 模型 | 表现 | 注意事项 |
|------|------|---------|
| Claude 4 | 原生多模态，视觉理解和推理能力强；zero-shot Multimodal CoT 效果稳定 | 对低质量图像（截图截图、压缩失真）的推理可靠性下降，建议提供高分辨率图 |
| GPT-4V / GPT-4o | 视觉推理能力强，支持多图对比；CoT 格式响应稳定 | 图像 token 消耗较多，成本比纯文本高；large 模式精度更高但更贵 |
| Gemini 2 | 原生多模态，支持视频帧、长图等复杂视觉输入 | 某些图表类型（手绘草图、低对比度图）识别率较低，建议在 prompt 中补充文字描述辅助 |
| 小型开源多模态模型（LLaVA 等） | CoT 效果较差，推理步骤常常流于表面 | 建议用 few-shot 给足示例，或降低对推理深度的期望 |

## 常见踩坑

**只要求描述图像，没有明确推理任务** — "请描述这张图" 得到的是感知输出，不是推理。要加上具体问题和"请展示推理过程"，才能触发 Multimodal CoT。

**图像信息和文本指令矛盾** — 图中数据和 prompt 里的说明不一致，模型会产生混乱输出。以图为准，不要在 prompt 里重新描述图像内容（除非是补充图像看不到的背景信息）。

**推理步骤停在感知层** — 模型只描述了"图中有 A 和 B"，没有进行真正的推理。避免方式：明确要求"基于以上观察，推断/计算/判断..."，把推理步骤单独分出来。

**多图时不指定图像顺序** — 如果 prompt 里有多张图，不标注"图1/图2"，模型可能混淆来源。始终给图像编号并在问题中明确引用。

**对推理过程的正确性过度信任** — Multimodal CoT 提升了推理可见性，但不保证推理正确。中间步骤仍可能出错，高风险场景需要人工审核推理链。

## 来源

- Zhang, Z. et al. (2023). *Multimodal Chain-of-Thought Reasoning in Language Models*. TMLR 2023. https://arxiv.org/abs/2302.00923
- Wei, J. et al. (2022). *Chain-of-Thought Prompting Elicits Reasoning in Large Language Models*. NeurIPS 2022. https://arxiv.org/abs/2201.11903
- Prompt Engineering Guide (DAIR.AI). Multimodal CoT Prompting. https://www.promptingguide.ai/techniques/multimodalcot
- 关联 wiki: [[wiki/prompt-system]]
