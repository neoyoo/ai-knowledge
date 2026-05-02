---
title: "evaluation-observability——agentscope"
category: L2
parent: "[[evaluation-observability]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "evaluation-observability"
created: "2026-04-15"
confidence: high
---

## 概述

AgentScope 把「运行可观测」和「效果评估」设计为两个独立但深度耦合的模块：`tracing/` 基于 OpenTelemetry 标准为 Agent/LLM/Tool/Embedding/Formatter 五类调用层分别提供装饰器级别的 span 注入；`evaluate/` 则提供 Benchmark-Task-Metric-Evaluator-Storage 五层评估管道，并通过 OpenTelemetry Baggage + `_InMemoryExporter` 把运行时追踪数据直接桥接到评估统计，实现"跑评测时自动采集 token 用量"的闭环。

---

## 架构分析

### Tracing 层：OpenTelemetry 装饰器矩阵

`tracing/` 的核心设计是针对不同调用层定制化的装饰器，而非通用 AOP 拦截。共五个专用装饰器：

| 装饰器 | 被装饰目标 | 关键 span 属性 |
|--------|-----------|----------------|
| `trace_reply` | `AgentBase.reply()` | `gen_ai.operation.name=invoke_agent`、agent id/name/description、输入输出 Msg |
| `trace_llm` | `ChatModelBase.__call__()` | `gen_ai.operation.name=chat`、provider/model/温度等参数、token usage、输出 parts |
| `trace_toolkit` | `Toolkit.call_tool_function()` | `gen_ai.operation.name=execute_tool`、tool_call_id/name/description/arguments |
| `trace_embedding` | `EmbeddingModelBase.__call__()` | `gen_ai.operation.name=embeddings`、model/维度数 |
| `trace_format` | `FormatterBase.__call__()` | `gen_ai.operation.name=format`、format target（provider 名称）、输出消息数 |
| `trace` | 任意 async/sync 函数 | `gen_ai.operation.name=invoke_generic_function`、函数名/输入输出 |

所有 span 均附加 `gen_ai.conversation_id`（即 AgentScope 的 `run_id`）作为会话关联键。

**属性命名策略**：最大程度复用 OpenTelemetry GenAI Semantic Conventions（`opentelemetry-semantic-conventions` 包的 `gen_ai_attributes`），仅在框架特有场景追加 `agentscope.*` 前缀扩展属性（如 `agentscope.format.target`、`agentscope.function.name`、`agentscope.function.input/output`）。

### Tracing 初始化：按需激活

```python
# tracing/_setup.py
def setup_tracing(endpoint: str) -> None:
    exporter = OTLPSpanExporter(endpoint=endpoint)
    span_processor = BatchSpanProcessor(exporter)
    tracer_provider = TracerProvider()
    tracer_provider.add_span_processor(span_processor)
    trace.set_tracer_provider(tracer_provider)

def _get_tracer() -> Tracer:
    return trace.get_tracer("agentscope", __version__)
```

`_check_tracing_enabled()` 检查全局 `_config.trace_enabled` 标志；未调用 `setup_tracing()` 时装饰器完全透明（zero-overhead bypass）。

### Evaluate 层：五层管道

```
BenchmarkBase          → 数据集抽象，__iter__ 产出 Task
    └── Task           → id + input + ground_truth + metrics + tags + metadata
            └── MetricBase(async __call__)  → 返回 MetricResult
EvaluatorBase.run(solution)
    ├── GeneralEvaluator  → 串行，适合调试
    └── RayEvaluator      → Ray Actor 并发，适合大规模评测
EvaluatorStorageBase   → 持久化接口
    └── FileEvaluatorStorage → 文件系统实现
```

**solution 签名约定**：`async (task: Task, pre_hook: Callable) -> SolutionOutput`。`SolutionOutput` 含 `success/output/trajectory/meta`，其中 `trajectory` 是完整的 `ToolUseBlock | ToolResultBlock | TextBlock` 序列，供过程准确率指标使用。

### 评估-追踪桥接：`_InMemoryExporter` + OpenTelemetry Baggage

评估执行时，`GeneralEvaluator.run_solution()` 和 `RaySolutionActor.run()` 都会：

1. 通过 Baggage 注入 `task_id` 和 `repeat_id`
2. 打开根 span `Solution_{task_id}_{repeat_id}`，并把 `_config.trace_enabled` 置 `True`
3. `_InMemoryExporter.export()` 从 Baggage 读取 task/repeat 上下文，按 span 的 `gen_ai.operation.name` 分类计数：LLM 调用次数 + token 用量、Agent 调用次数、Tool 调用次数（按名）、Embedding 调用次数
4. 评测结束后把统计数据写入存储（`stats.json`），在 `aggregate()` 阶段合并到全局报告

这使得"评测时消耗了多少 token/调了多少工具"完全自动采集，无需在 solution 代码中埋点。

### ACEBenchmark：内置基准

`_ace_benchmark/` 是对 [ACEBench](https://github.com/ACEBench/ACEBench) 的官方集成，专注于中文手机 Agent 的工具调用能力评估。两个内置 Metric：

- `ACEAccuracy`：最终状态匹配（工具调用后手机状态与 ground truth 逐键比对）
- `ACEProcessAccuracy`：里程碑路径匹配（trajectory 中是否包含所有必要的工具调用序列）

### FileEvaluatorStorage：可续跑的文件系统存储

目录结构：
```
save_dir/
├── evaluation_meta.json      # 评测元信息
├── evaluation_result.json    # 聚合结果
├── {task_id}/
│   ├── task_meta.json
│   └── {repeat_id}/
│       ├── solution.json     # SolutionOutput
│       ├── stats.json        # 运行时统计（来自 _InMemoryExporter）
│       ├── logging.txt       # Agent 打印日志（pre_hook 捕获）
│       └── evaluation/
│           └── {metric_name}.json  # MetricResult
```

`solution_result_exists()` 在每次 `run_solution()` 前检查——文件已存在则跳过，天然支持断点续评。

---

## 关键代码路径

### 路径 1：LLM 调用追踪

```
ChatModelBase.__call__()
  [被 @trace_llm 装饰]
    → _check_tracing_enabled()          # tracing/_trace.py:69
    → _get_tracer()                     # tracing/_setup.py:41
    → _get_llm_request_attributes()     # tracing/_extractor.py:198
        ├── _get_provider_name()        # 类名 + base_url 双重映射
        └── _get_tool_definitions()     # OpenAI nested → OTel flat 格式转换
    → tracer.start_as_current_span()
    → await func(...)                   # 真实 LLM 调用
    → _get_llm_response_attributes()    # tracing/_extractor.py:346
        └── _get_llm_output_messages()  # ChatResponse.content → parts 格式
    → _set_span_success_status() / _set_span_error_status()
    # 流式返回时 → _trace_async_generator_wrapper()，在最后一个 chunk 后写响应属性
```

### 路径 2：Agent 回复追踪

```
AgentBase.reply()
  [被 @trace_reply 装饰]
    → _get_agent_request_attributes()   # tracing/_extractor.py:447
        └── _get_agent_messages()       # Msg.get_content_blocks() → parts 格式
    → tracer.start_as_current_span("invoke_agent {agent_name}")
    → await func(self, ...)
    → _get_agent_response_attributes()  # tracing/_extractor.py:526
    → _set_span_success_status()
```

### 路径 3：评测全流程（GeneralEvaluator）

```
GeneralEvaluator.run(solution)
  → 初始化 _InMemoryExporter + SimpleSpanProcessor
  → _save_evaluation_meta()
  → for task in benchmark:
      _save_task_meta(task)
      for repeat_id in range(n_repeat):
          run_solution(repeat_id, task, solution)
            → baggage.set_baggage("task_id", task.id)
            → baggage.set_baggage("repeat_id", repeat_id)
            → _config.trace_enabled = True
            → tracer.start_as_current_span("Solution_{task_id}_{repeat_id}")
            → solution(task, pre_print_hook)  # 用户提供的 agent 逻辑
            → storage.save_solution_result()
            → storage.save_solution_stats(exporter.cnt[task_id][repeat_id])
          for metric in task.metrics:
              run_evaluation(task, repeat_id, solution_output)
                → task.evaluate(solution_output)
                    → [metric(solution_output) for metric in task.metrics]
                → storage.save_evaluation_result(result)
  → aggregate()
      → storage.get_solution_stats() → 汇总 llm/agent/tool/embedding/token 用量
      → storage.save_aggregation_result()
```

### 路径 4：ACEAccuracy 指标计算

```
ACEAccuracy.__call__(solution: SolutionOutput)
  → solution.output  # 最终手机状态列表
  → self.state       # ground truth 状态列表
  → 逐键比对（处理 ACEBench 数据集中的 API/Api 拼写错误）
  → 返回 MetricResult(name="accuracy", result=0|1)

ACEProcessAccuracy.__call__(solution: SolutionOutput)
  → solution.trajectory  # ToolUseBlock 序列
  → 转换为 "[func(arg='v')]" 字符串格式
  → 检查每个 mile_stone 是否存在于 trajectory 中
  → 返回 MetricResult(name="process_accuracy", result=0|1)
```

---

## 设计亮点

### 1. OpenTelemetry 标准 + 框架扩展属性的双层命名

属性键优先使用 OTel GenAI Semantic Conventions（`GenAIAttributes.GEN_AI_*`），确保与 Jaeger/Grafana/Langfuse 等标准工具直接兼容。框架特有信息用 `agentscope.*` 前缀追加，不污染标准命名空间。这使得 AgentScope trace 数据无需适配层即可接入行业标准 APM。

### 2. 评估-追踪深度耦合：_InMemoryExporter + Baggage 桥接

评测框架不是在 solution 代码外手动统计调用次数，而是复用 tracing 层的 span 数据。`_InMemoryExporter` 实现 OTel `SpanExporter` 接口，从 Baggage 中读取 `task_id/repeat_id` 上下文，自动把每个 span 路由到对应的统计桶。这样同一套代码既可连接外部 OTLP 后端（用于生产观测），也可在评估时切换为内存导出（用于指标统计）。

### 3. 流式响应的 span 生命周期管理

流式 LLM 响应不能在调用发起时立即写属性，AgentScope 通过 `_trace_async_generator_wrapper` 和 `_trace_sync_generator_wrapper` 包装 generator，在最后一个 chunk 产出后才写 response attributes 并结束 span。这正确保持了 span 的完整性，而非截断到首 token。

### 4. Provider 推断的双重策略（类名 + base_url）

`_get_provider_name()` 先检查类名前缀（`DashscopeChatModel` → `dashscope`），对 `OpenAIChatModel` 额外检查 `client.base_url`，从而把通过 OpenAI 兼容接口接入的 DeepSeek/Moonshot/Azure 等自动识别出正确的 provider 标签，避免所有兼容接口都被标记为 openai。

### 5. 可续跑的评测存储设计

`FileEvaluatorStorage` 在每次 `run_solution` 前检查 `solution.json` 是否已存在，已存在直接从文件加载跳过执行。评测可以随时中断后从断点继续，不浪费已完成的 API 调用。

### 6. 过程准确率（ProcessAccuracy）：不只看最终结果

`ACEProcessAccuracy` 检查 trajectory 中的里程碑调用序列，而非只看最终状态。这捕获了"结果对但路径错"的情况，比纯结果评估更能反映 agent 的推理合理性。

---

## 局限性

### 1. finish_reason 硬编码为 `"stop"`

`_get_llm_response_attributes()` 中有明确的 FIXME 注释：`# FIXME: finish reason should be capture in chat response`，当前所有 LLM 响应的 `gen_ai.response.finish_reasons` 固定为 `'["stop"]'`，不反映实际的 `tool_use`/`length` 等终止原因。

### 2. 流式追踪的最终 chunk 丢失风险

`_trace_async_generator_wrapper` 在 `finally` 块中写属性，使用的是 `last_chunk` 变量，若 generator 在第一个 chunk 前就抛异常，`last_chunk` 为 `None`，response attributes 会写入 `None` 值而非完全跳过（虽然有错误状态保护，但属性可能部分缺失）。

### 3. `_InMemoryExporter` 与评测上下文强耦合

`_InMemoryExporter` 依赖 OpenTelemetry Baggage 获取 `task_id/repeat_id`，这要求评测代码必须在正确的 Baggage 上下文内运行。若在多线程环境下 Baggage 传播不正确，统计数据会路由失败（静默丢弃：`if task_id is None or repeat_id is None: continue`）。

### 4. ACEBenchmark 仅支持中文数据集

代码中 `data_subdir = ["data_zh"]`，英文版本（`data_en`）已注释，说明英文评测能力尚不完整。

### 5. 评估管道无并行 Metric 计算（GeneralEvaluator）

`GeneralEvaluator.run_evaluation()` 串行 `await metric(solution)` for 循环，多个 Metric 不并行，大型 benchmark + 多 Metric 场景下效率较低（RayEvaluator 用 `asyncio.gather(futures)` 解决了这个问题，但需要 Ray 依赖）。

### 6. tracing 装饰器只覆盖 async 路径（部分）

`trace_reply`/`trace_llm`/`trace_format`/`trace_toolkit`/`trace_embedding` 均要求被装饰函数是 async 的；同步调用只有通用 `trace` 装饰器支持，若用户自定义同步模型不走 AgentScope 标准类，需手动添加。

---

## 来源

- 源码版本：`0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12`
- 分析深度：源码级
- 核心文件：
  - `src/agentscope/tracing/_trace.py` — 装饰器实现
  - `src/agentscope/tracing/_extractor.py` — span 属性提取
  - `src/agentscope/tracing/_attributes.py` — 属性命名常量
  - `src/agentscope/tracing/_setup.py` — TracerProvider 初始化
  - `src/agentscope/tracing/_converter.py` — ContentBlock → OTel parts 转换
  - `src/agentscope/evaluate/_evaluator/_general_evaluator.py` — 串行评测流程
  - `src/agentscope/evaluate/_evaluator/_ray_evaluator.py` — Ray 并发评测
  - `src/agentscope/evaluate/_evaluator/_in_memory_exporter.py` — 追踪桥接
  - `src/agentscope/evaluate/_evaluator_storage/_file_evaluator_storage.py` — 持久化存储
  - `src/agentscope/evaluate/_ace_benchmark/_ace_benchmark.py` — ACEBench 集成
  - `src/agentscope/evaluate/_ace_benchmark/_ace_metric.py` — 准确率指标实现
