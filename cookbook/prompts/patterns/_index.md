# Prompt Patterns

> 核心提示词设计模式。每个 pattern 说明什么时候用、什么时候不用、怎么用、和其他 pattern 怎么组合。

## 按类别浏览

### Reasoning（推理）
- [Chain-of-Thought](chain-of-thought.md) — 引导模型分步推理，提升复杂任务准确率
- [Self-Consistency](self-consistency.md) — 多次采样取多数，提高推理可靠性（计划中）
- [Tree of Thoughts](tree-of-thoughts.md) — 树状搜索多条推理路径（计划中）

### Action（行动）
- [ReAct](react.md) — 推理与行动交替循环，agent 核心运行模式
- [Reflexion](reflexion.md) — ReAct + 失败后自我反思重试（计划中）

### Output（输出控制）
- [Structured Output](structured-output.md) — 约束输出为 JSON/XML，支撑 tool calling（计划中）
- [Few-shot Prompting](few-shot.md) — 用示例引导输出格式和风格（计划中）

### Orchestration（编排）
- [Prompt Chaining](prompt-chaining.md) — 将复杂任务拆为 prompt 流水线（计划中）
- [RAG](rag.md) — 检索增强生成，注入外部知识（计划中）

### Meta（元设计）
- [System Prompt Design](system-prompt-design.md) — 系统提示词结构化设计（计划中）
- [Meta Prompting](meta-prompting.md) — 用 LLM 生成或优化 prompt（计划中）

### Specialized（专用）
- [Zero-shot Prompting](zero-shot.md) — 无示例直接指令（计划中）
- [Generate Knowledge](generate-knowledge.md) — 先生成知识再回答（计划中）
- [PAL](pal.md) — 用代码辅助推理（计划中）
- [ART](art.md) — 自动推理与工具使用（计划中）
- [Directional Stimulus](directional-stimulus.md) — 定向刺激引导（计划中）
- [Graph Prompting](graph-prompting.md) — 图结构推理（计划中）
- [Multimodal CoT](multimodal-cot.md) — 多模态推理（计划中）
- [Active-Prompt](active-prompt.md) — 主动选择示例（计划中）
- [APE](ape.md) — 自动 prompt 工程（计划中）

## 怎么选

- 需要模型推理 → Chain-of-Thought
- 需要模型调用工具 → ReAct + Structured Output
- 任务太复杂一个 prompt 搞不定 → Prompt Chaining
- 需要注入外部知识 → RAG
- 搭一个新 agent → System Prompt Design（第一步）
- 需要稳定输出格式 → Structured Output + Few-shot
