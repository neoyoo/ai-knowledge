# Prompt Templates

> 按场景分类的可直接复制模板。每个模板带 {{placeholder}}，填入你的参数即可使用。

## 模板列表

### Agent 构建
- [Agent System Prompt](agent-system-prompt.md) — 生产级 Agent 的完整 system prompt 模板，覆盖身份、能力、工作流、规则、错误处理、输出格式七个模块，直接填充即可上线。
- [Tool Use Agent](tool-use-agent.md) — 为 Agent 配备工具调用能力的完整模板，涵盖系统提示词、Anthropic 和 OpenAI 两种工具定义格式、多工具路由策略、工具失败处理。
- [Multi-Agent Orchestrator](multi-agent-orchestrator.md) — 把复杂任务交给一个 Orchestrator 分解、分派给多个 Worker Agent 并行执行，最后汇总结果。
- [Conversation Agent](conversation-agent.md) — 构建有角色、有记忆、能保持多轮上下文的对话 Agent，适配从客服机器人到个人助手的各类场景。

### 数据处理
- [RAG QA](rag-qa.md) — 检索结果注入 prompt，让模型只基于真实文档回答，附来源编号，不确定时说"不知道"。
- [Data Extraction](data-extraction.md) — 从非结构化文本（合同、报告、日志、邮件）按 JSON Schema 提取结构化字段，缺失字段输出 null，不猜测。
- [Summarization](summarization.md) — 把长文档或多轮对话压缩成指定长度和格式的摘要，从一句话 TL;DR 到分层结构摘要，适配不同消费场景。
- [Classification](classification.md) — 把文本输入映射到预定义类别，从简单的单标签分类到带置信度的多标签路由，覆盖 agent 中最常见的意图识别和内容分发场景。

### 质量评估
- [Code Review](code-review.md) — 让 LLM 扮演资深工程师，对提交的代码给出分级、结构化的审查意见，覆盖正确性、安全、性能、可维护性四个维度。
- [Evaluation Judge](evaluation-judge.md) — 用一个 LLM 作为评估器，对另一个 LLM 的输出打分或做对比，替代部分人工标注，实现自动化质量评估。

## 怎么选
- 从零搭 Agent → Agent System Prompt
- Agent 需要调工具 → Tool Use Agent
- 多 Agent 协作 → Multi-Agent Orchestrator
- 基于知识库问答 → RAG QA
- 从文本提取结构化数据 → Data Extraction
- 需要代码审查 → Code Review
- 评估模型输出质量 → Evaluation Judge
