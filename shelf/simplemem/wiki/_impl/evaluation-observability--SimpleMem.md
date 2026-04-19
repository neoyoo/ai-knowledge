---
title: "evaluation-observability——SimpleMem"
category: L2
parent: "[[evaluation-observability]]"
source: SimpleMem
source_version: "94ef7d76786af96878dea6e87ea2c7f5eaeae168"
concept: evaluation-observability
created: "2026-04-15"
updated: "2026-04-15"
confidence: medium
---

## 概述

SimpleMem 的评估和可观测性主要在 OmniSimpleMem 层实现，`OmniSimpleMem/omni_memory/evaluation/` 包含基准测试框架和指标系统。cross/ 层通过 `collectors.py` 的 `ObservationExtractor` 实现运行时行为提取（非性能 metrics）。文本版核心层（core/）缺少内建 metrics，但提供了 `test_locomo10.py` 等评估脚本用于基准对比。

## 架构分析

### 评估框架（OmniSimpleMem/omni_memory/evaluation/）

```
evaluation/
├── __init__.py
├── benchmarks.py    # 基准测试套件
├── evaluator.py     # 评估器主类
└── metrics.py       # 指标定义和计算
```

`evaluator.py` 的 `MemoryEvaluator` 负责端到端评估：输入 QA pairs，运行记忆系统，对比预测答案与 ground truth，输出精度/召回/F1 等指标。

`benchmarks.py` 集成标准基准数据集（如 LoCoMo），支持批量评估。

`metrics.py` 定义核心指标：
- **Exact Match（EM）**：精确字符串匹配
- **F1 Score**：基于 token 级别的 F1
- **Hit Rate@K**：检索命中率
- **Latency**：端到端响应延迟（ms）

### 跨版本基准测试脚本

- `test_locomo10.py`：基于 LoCoMo-10 数据集的基准测试脚本，支持 text 版和 omni 版对比
- `tests/` 目录：单元测试 + 集成测试

### 运行时可观测性（cross/collectors.py）

`ObservationExtractor` 实现行为级可观测：
- 从 `SessionEvent` 流中 LLM 提取结构化观察（decision/bugfix/feature/discovery 等）
- 每条 `CrossObservation` 带 type 分类，可按类型统计 session 行为分布
- `cross_session_stats` MCP tool 暴露系统级统计（session 数/event 数/observation 数/memory entry 数）

### Stats API

```python
# cross/api_mcp.py: cross_session_stats tool
{
    "sessions_total": N,          # 总会话数
    "sessions_active": M,         # 当前活跃会话数
    "events_total": K,            # 总事件数
    "observations_total": J,      # 总观察数
    "memory_entries_total": L     # 总记忆条目数
}
```

## 关键代码路径

### 端到端评估（evaluator.py）

```python
# OmniSimpleMem/omni_memory/evaluation/evaluator.py
class MemoryEvaluator:
    def evaluate(self, test_cases: List[TestCase]) -> EvaluationReport:
        results = []
        for case in test_cases:
            # 写入记忆
            self.memory.add_text(case.context)
            # 检索回答
            pred = self.memory.query(case.question, top_k=10)
            # 计算指标
            em = compute_exact_match(pred.answer, case.ground_truth)
            f1 = compute_f1(pred.answer, case.ground_truth)
            results.append(EvalResult(em=em, f1=f1, latency_ms=pred.latency))
        
        return EvaluationReport(
            total=len(results),
            em_mean=mean(r.em for r in results),
            f1_mean=mean(r.f1 for r in results),
            latency_p50=percentile(50, [r.latency_ms for r in results]),
            latency_p95=percentile(95, [r.latency_ms for r in results]),
        )
```

### LoCoMo 基准测试（test_locomo10.py）

```python
# test_locomo10.py (项目根目录)
# 加载 LoCoMo-10 数据集 (10 multi-session conversation samples)
# 运行 SimpleMem text 版
# 输出: EM, F1, Recall@5, 平均延迟
```

README 中记录的 benchmark 结果：
- SimpleMem vs Mem0: EM +47%，F1 +51%（LoCoMo 数据集）
- OmniSimpleMem vs 基线：多模态场景显著提升

### ObservationExtractor 提取（collectors.py）

```python
class ObservationExtractor:
    def extract(self, events: List[SessionEvent]) -> List[CrossObservation]:
        # 过滤 full redaction 事件
        visible_events = [e for e in events if e.redaction_level != RedactionLevel.full]
        event_text = "\n".join([f"[{e.kind}] {e.title}: {e.payload_json}" for e in visible_events])
        
        prompt = f"""
        Extract structured observations from this session:
        {event_text}
        
        For each significant event, output JSON:
        {{"type": "decision|bugfix|feature|refactor|discovery|change", "title": "...", "narrative": "..."}}
        """
        response = llm.chat_completion([...])
        return parse_observations_json(response)
```

## 设计亮点

1. **LoCoMo 权威基准**：采用 LoCoMo 多会话对话数据集（而非自建测试集），评估结果可与学术界对比，+47%/+51% 的 EM/F1 提升有可重现验证路径。

2. **ObservationType 分类统计**：6 种 observation 类型（decision/bugfix/feature/refactor/discovery/change）使开发者可以按类型分析 agent 行为分布（如"本月做了多少 bugfix"），超出了普通 tracing 的能力。

3. **Stats MCP Tool**：`cross_session_stats` 将系统健康指标直接暴露为 MCP tool，agent 框架可定期轮询用于监控告警，无需额外 monitoring infrastructure。

4. **Latency 分位数**：Evaluator 记录 P50/P95 延迟而非仅均值，对延迟分布不均匀（有长尾的 LLM 调用）场景更具指导价值。

## 局限性

1. **文本版缺少内建 metrics**：`core/` 的 `SimpleMemSystem` 没有任何 prometheus metrics、logging 结构化输出或 tracing hooks，只有 print 语句，生产可观测性很弱。

2. **ObservationExtractor 质量无量化**：LLM 提取的 observation 质量（准确性、完整性）本身没有评估机制，用户无法知道有多少重要事件被遗漏。

3. **评估框架仅在 OmniSimpleMem 层**：`evaluation/` 目录仅在 OmniSimpleMem 中，文本版 `SimpleMemSystem` 的精度评估只能通过根目录的 `test_locomo10.py` 脚本手动运行，无集成评估 API。

4. **无分布式追踪（Tracing）**：没有 OpenTelemetry/Jaeger 集成，无法追踪单次 `ask()` 调用中 planning/retrieval/reflection 各阶段的耗时分布，难以定位性能瓶颈。

## 来源

- 源码版本：94ef7d76786af96878dea6e87ea2c7f5eaeae168
- 分析深度：源码级
