---
title: Finetuning System
aliases: [微调系统, fine-tuning, model training, LLM training, prompt optimization]
category: L1
tags: [finetuning, training, RL, SFT, DPO, GRPO, prompt-optimization]
sources: [agentscope]
created: 2026-04-15
updated: 2026-04-15
relations:
  - target: "[[evaluation-observability]]"
    type: feeds
  - target: "[[tool-system]]"
    type: uses
  - target: "[[prompt-system]]"
    type: alternative
  - target: "[[query-loop]]"
    type: uses
---

## 一句话定义

在 agent 部署前通过算法（GRPO/SFT）或自动优化（prompt tuning）改变模型行为，使其在特定任务上超越通用基线。

## 核心问题

- 什么时候应该微调模型，什么时候优化 prompt 就够了？
- 不同微调算法（GRPO/SFT/DPO）各自适合什么场景？
- 如何生成高质量的训练数据？
- 如何评估微调效果，避免在错误指标上过拟合？
- 微调的基础设施成本和工程复杂度是否值得？

## 各家对比

| 维度 | AgentScope | （待扩源）|
|------|-----------|----------|
| 核心设计 | `tuner/` 模块作为独立训练管道：GRPO/SFT 对接 Trinity-RFT 框架执行强化学习与监督微调，DSPy MIPROv2 负责 prompt 自动优化，两条路径在同一 `Tuner` 抽象下统一管理 | — |
| 训练算法 | GRPO（Group Relative Policy Optimization，无需 critic 网络的 PPO 变体）+ SFT（监督微调），通过 `backend: trinity-rft` 配置切换；GRPO 适合 reward 信号驱动的任务，SFT 适合有标注样本的任务 | — |
| Prompt 优化 | DSPy MIPROv2：将 prompt 模板参数化，通过贝叶斯优化搜索最优 few-shot 示例和指令前缀，无需梯度更新，只需评估函数和少量标注样本 | — |
| 模型选择 | `ModelSelector`：基于任务描述 + 评估指标自动从候选模型池挑选基础模型，避免人工枚举对比 | — |
| 训练数据 | `DataGenerator`：通过 agent 与工具交互自动生成训练轨迹，可配置采样策略，配合 `Evaluator` 过滤低质样本 | — |
| 评估集成 | `Evaluator` 模块：自定义评估函数 + 多指标聚合，训练循环中在线评估，防止在单一指标上过拟合 | — |
| 适用场景 | 任务特化（代码生成/结构化输出/特定领域问答）、当前模型准确率不达标、有充足标注数据或可自动生成训练数据 | — |
| 主要局限 | 依赖外部框架（Trinity-RFT、DSPy），无法独立运行；GRPO reward 函数设计难度高；训练基础设施成本（GPU）不低 | — |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表实现 |
|------|---------|---------|---------|
| 方案 A：Prompt 优化（DSPy MIPROv2） | 保持模型权重不变，自动搜索最优 prompt 模板 + few-shot 示例 | 有评估函数但缺乏大量标注数据；需要快速迭代；不想负担 GPU 成本 | AgentScope DSPy tuner |
| 方案 B：SFT（监督微调） | 在高质量标注样本上直接梯度更新，拟合目标行为分布 | 有 500+ 高质量标注样本；目标行为明确可描述；需要稳定一致的输出格式 | AgentScope Trinity-RFT SFT |
| 方案 C：GRPO/强化学习 | 通过奖励信号引导模型行为，无需大量人工标注 | 可以定义明确的 reward 函数（准确率/格式/通过率）；标注数据稀缺；需要模型学会策略而非记忆样本 | AgentScope Trinity-RFT GRPO |
| 方案 D：DPO（偏好优化） | 从人类偏好对（好/差）学习，比 RLHF 更稳定 | 有偏好标注数据；对话质量/风格对齐场景 | 多数 alignment 场景 |
| 方案 E：不微调，只优化 prompt | 在 system prompt / few-shot 上花功夫 | 任务不需要权重级别的知识注入；预算有限；快速验证阶段 | 通用最佳实践 |

### 场景决策指南

**如果你的问题是"模型不理解任务格式/风格" → 先试方案 A（Prompt 优化）**
- 原因：大多数"模型不好用"的问题本质是 prompt 没写好，DSPy MIPROv2 自动搜索 few-shot 组合的成本远低于微调；先跑 prompt 优化，确认上限后再考虑微调
- 注意：MIPROv2 需要提供评估函数，评估函数写不好（过于简单或依赖 LLM-as-judge）会产生虚假进步；每轮优化需要 20-100 次模型调用，latency 不低

**如果你有大量高质量标注数据（>500 样本）且目标行为明确 → 选方案 B（SFT）**
- 原因：SFT 是最直接的方式——给模型看足够多的"正确答案"，它就会模仿；数据质量比数量更关键，100 条精标数据 > 1000 条半自动数据
- 注意：SFT 容易对训练集过拟合（catastrophic forgetting）；需要定期在保留集评估通用能力退化情况；避免数据分布与实际使用偏差过大

**如果你能定义明确的 reward 函数但缺标注数据 → 选方案 C（GRPO）**
- 原因：GRPO 无需 critic 网络，比传统 PPO 更稳定，适合可程序化验证的任务（代码运行通过、数学答案正确、JSON 格式合规）；AgentScope 的 `DataGenerator` 可自动采集训练轨迹降低数据生成成本
- 注意：reward hacking 是主要风险——模型会找到绕过 reward 函数的捷径而非真正学会任务；reward 函数必须足够鲁棒；训练过程需要持续监控分布偏移

**如果你需要对齐对话风格/偏好而非能力 → 选方案 D（DPO）**
- 原因：DPO 直接从偏好对学习，避免了 PPO/GRPO 的 reward 设计难题；对话质量、拒绝风格、安全对齐等场景天然产生偏好数据
- 注意：偏好数据的多样性至关重要，单一评注者的偏见会被放大；需要注意 chosen/rejected 样本分布，避免 chosen 样本太短导致模型倾向截断回答

**如果目标是在有限预算内快速提升特定任务准确率 → Prompt 优化优先，微调作为保底**
- 先用方案 A 确定 prompt 优化的天花板（通常能提升 10-30%），如果还差 5 个百分点再上微调
- AgentScope 的 `ModelSelector` 可在微调前自动筛选最适合任务的基础模型，避免在错误的基础上微调

### 常见陷阱

- **跳过 prompt 优化直接上微调**：微调的工程成本（GPU、数据标注、训练管道）比 prompt 优化高 10-100 倍；80% 的任务通过精心设计的 prompt + few-shot 就能达标，先做廉价实验再决策是否微调
- **用 LLM-as-judge 作为唯一训练 reward**：如果 reward 本身来自另一个 LLM，模型会学会欺骗 judge 而非完成真实任务（Goodhart's Law）；必须配合程序化可验证指标（准确率/格式合规/运行结果）
- **不分离训练集/评估集**：在同一数据上生成训练数据和评估，导致虚假进步；DataGenerator 生成的数据必须在创建时就划分 hold-out set，不能事后分割
- **忽视通用能力退化**：微调几乎总是以损失一定通用能力为代价；SFT/GRPO 后必须在 MMLU/HumanEval 等通用基准上评估退化幅度，确认代价可接受
- **reward hacking 不监控**：模型在训练中发现了 reward 函数的漏洞（如只要回答长就能得高分），导致验证集指标高但实际使用效果差；训练过程中需要人工抽样检查模型输出，而不只看数字
- **不考虑基础设施成本就做微调计划**：GRPO 需要多 GPU 训练（通常 4-8 块 A100），Trinity-RFT 这类框架对环境依赖复杂；在确认任务价值和数据可行性前，不应启动微调基础设施建设

## 关系

- [[evaluation-observability]]：微调效果的度量依赖评估体系；`Evaluator` 模块的指标设计与评估可观测性直接相关
- [[tool-system]]：`DataGenerator` 通过 agent 与工具交互自动生成训练轨迹，工具系统是训练数据的生产设施
- [[prompt-system]]：Prompt 优化（方案 A）是微调的替代方案，二者在提升模型效果上形成互补——prompt 优化无需修改权重，微调改变权重但保持 prompt 简单
- [[query-loop]]：训练数据生成过程本身是一个 agent loop（采样 → 执行 → 评估 → 过滤）；训练好的模型最终服务于 agent loop 中的推理节点

## 来源

| 项目 | 版本 | 分析深度 | 备注 |
|------|------|---------|------|
| agentscope | v0.1.x（2024） | L2 详情 | 阿里巴巴出品，tuner/ 模块包含完整 GRPO/SFT/DSPy 管道 |

## L2 详情

- [[finetuning-system--agentscope]]
