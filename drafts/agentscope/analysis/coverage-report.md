# 覆盖率扫描报告：agentscope

**生成时间**: 2026-04-15

## 汇总

| 指标 | 数值 |
|------|------|
| 有效顶层模块总数 | 21 |
| 已覆盖模块数 | 19 |
| 覆盖率 | 19/21 = 90.5% |
| 状态 | PASS（≥80%）|
| 补漏轮次 | 0 |

## 模块覆盖详情

| 模块路径 | 对应 L2 | 状态 |
|---------|---------|------|
| agent/ | query-loop--agentscope.md | COVERED |
| pipeline/ | query-loop--agentscope.md | COVERED |
| model/ | query-loop--agentscope.md, prompt-system--agentscope.md | COVERED |
| formatter/ | prompt-system--agentscope.md | COVERED |
| memory/ | memory-system--agentscope.md | COVERED |
| rag/ | memory-system--agentscope.md, context-management--agentscope.md | COVERED |
| embedding/ | memory-system--agentscope.md | COVERED |
| token/ | context-management--agentscope.md | COVERED |
| message/ | runtime-state--agentscope.md | COVERED |
| session/ | runtime-state--agentscope.md, session-recovery--agentscope.md | COVERED |
| hooks/ | hooks--agentscope.md | COVERED |
| mcp/ | mcp-skills--agentscope.md | COVERED |
| tool/ | tool-system--agentscope.md | COVERED |
| a2a/ | multi-agent--agentscope.md | COVERED |
| plan/ | multi-agent--agentscope.md | COVERED |
| realtime/ | channel-remote--agentscope.md | COVERED |
| tts/ | channel-remote--agentscope.md | COVERED |
| tracing/ | evaluation-observability--agentscope.md | COVERED |
| evaluate/ | evaluation-observability--agentscope.md | COVERED |
| tune/ | （已废弃模块，重定向到 tuner/） | UNCOVERED |
| tuner/ | （无对应 L1 概念，ontology 无 fine-tuning 维度） | UNCOVERED |

## 排除模块

| 路径 | 排除原因 |
|------|---------|
| _utils/ | 纯工具函数，非架构概念 |
| exception/ | 异常定义，非架构概念 |
| types/ | 类型定义文件，非架构概念 |
| module/ | 模块注册基础设施，非独立概念 |
| docs/ | 文档，非源码 |
| tests/ | 测试，非源码 |
| examples/ | 示例，非源码 |
| assets/ | 静态资源，非源码 |

## 概念交叉检查发现

| 概念维度 | 关键词 | 发现 | 处理 |
|---------|-------|------|------|
| memory-system | dream, archive, consolidate, compact | 无匹配 | 无遗漏，AgentScope 不实现这些模式 |
| memory-system | recall | `_reme_task_long_term_memory.py` 中有 recall 语义注释（描述使用场景），非独立功能接口 | 已覆盖于 memory-system--agentscope.md 的 ReMe 长期记忆章节 |
| memory-system | forget | 仅出现在 evaluate 模块的字符串常量中，与记忆遗忘无关 | 无需处理 |
| memory-system | compress | `memory/_working_memory/_base.py` 中有 `_compressed_summary` + `update_compressed_summary()`，手动压缩摘要机制 | 已覆盖于 memory-system--agentscope.md（明确指出「手动压缩方案，不自动触发」） |
| query-loop | loop, run, step, turn, cycle, execute, dispatch, agentic | 主要命中 tuner/ 模块（训练循环语义），以及 pipeline/ 中编排逻辑 | pipeline/ 已覆盖；tuner/ 是训练领域，不属于推理 query-loop 概念 |
| tool-system | tool, call, invoke, register, executor, permission, sandbox | 主要命中 token/, tuner/, pipeline/, tracing/, types/_tool.py | token/ 已覆盖；tuner/ 为训练上下文的工具调用；tracing/ 已覆盖；tool/ 主模块已覆盖于 tool-system--agentscope.md |
| multi-agent | orchestrat, worker, spawn, delegate, coordinate, handoff, swarm | 仅命中 mcp/_stdio_stateful_client.py（coordinator 概念）、rag/_reader/（excel reader 无关）、evaluate/ 中的 ray_evaluator | mcp/ 已覆盖；ray_evaluator 的并发分发语义（`_ray_evaluator.py`）已涵盖于 evaluation-observability--agentscope.md |
| context-management | compress, summarize, window, token, truncate, prune | token/ 模块（5 种 token 计数器：anthropic/openai/gemini/huggingface/char）；memory/ 中 `_compressed_summary` | token/ 已覆盖于 context-management--agentscope.md；compress 已覆盖于 memory-system--agentscope.md |
| hooks | hook, event, lifecycle, pre_tool, post_tool, intercept, middleware | `types/_hook.py`、`agent/_react_agent.py`、`plan/_plan_notebook.py`、`realtime/_base.py`、`memory/_long_term_memory/` 相关文件 | hooks/ 主模块已覆盖于 hooks--agentscope.md；`plan/_plan_notebook.py` 中有 hook 调用，属 plan 流程钩子，已覆盖于 multi-agent--agentscope.md |

## tune/ 模块说明

`tune/` 是已废弃的遗留模块（`__init__.py` 直接抛出 `ImportError`，要求用户改用 `agentscope.tuner`）。

`tuner/` 是 AgentScope 的 agent fine-tuning 子系统，包含：
- `_tune.py`：主入口 `tune()` 函数，整合 workflow + judge + dataset + algorithm
- `_algorithm.py`：支持 `multi_step_grpo`、`sft` 等算法，对接 Trinity-RFT 框架
- `tuner/prompt_tune/`：基于 DSPy MIPROv2 的 prompt 自动优化
- `tuner/model_selection/`：基于 judge 函数的模型自动选择
- `_dataset.py`、`_judge.py`、`_model.py`：数据集、评判、模型配置

**未覆盖原因**：当前知识库 ontology 未设立「Agent Fine-Tuning / RL Training」维度，属于知识库范围外的训练系统。如需覆盖，建议在未来新增 L1 页面 `training-pipeline.md` 专门承载此维度。

## 补漏执行记录

本轮无需补漏。

19/21 模块已覆盖，未覆盖的 tune/tuner 属于知识库 ontology 范围外的训练系统（非推理时 agent 架构），不影响 ingest 完整性判断。
