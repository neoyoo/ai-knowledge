---
title: "EvolveMem 离线检索策略优化器"
category: insight
source: simplemem
source_version: "v0.3.0-2-g74174a1"
concept: evaluation-observability
created: 2026-06-09
updated: 2026-06-09
tags: [memory-system, evaluation-observability, retrieval, optimizer]
relations:
  - target: "[[evaluation-observability]]"
    type: supports
  - target: "[[memory-system]]"
    type: feeds
---

## 核心洞察

SimpleMem v0.3.0 最值得单独抽出的不是完整 memory runtime，而是 EvolveMem 的离线检索策略优化思路：用开发集问题评估当前 retrieval config，再诊断失败模式，提出参数/策略变更，并通过 guard 约束避免退化。

这把 memory recall 从“写死 top_k / fusion weights / budget”变成可评估、可迭代的配置搜索问题。

## 设计方案

EvolveMem 的闭环是：

```text
dev questions
  -> Evaluate current retrieval behavior
  -> Diagnose missing / noisy / over-budget recall
  -> Propose config mutation
  -> Guard against regression
  -> emit candidate retrieval config
```

`RetrievalConfig` 的 action space 覆盖：

- semantic / lexical / graph / symbolic 检索权重
- top-k 与 fusion strategy
- query decomposition
- answer verification
- MMR / diversity
- context budget

源码证据：

- `EvolveMem/README.md:27-31` — Evaluate / Diagnose / Propose / Guard 闭环
- `simplemem/evolver/evolution.py:4` — EvolutionEngine 的优化循环
- `simplemem/evolver/multi_retriever.py:39`、`:85`、`:137`、`:191` — `RetrievalConfig` action space

## 重要 caveat

当前 `simplemem.optimize()` 不是 production-ready optimizer：

- `simplemem/evolver/optimize.py:1-7` — 源码自称 degraded mode，不是 paper-faithful 实现
- `simplemem/evolver/optimize.py:97-105` — 当前只支持 text backend
- `simplemem/config.py:14` 与 `simplemem/text/system.py:26` — `Config` 输出尚未闭合接入 text backend 构造路径
- `simplemem/core/hybrid_retriever.py:47` — text retriever 仍读全局 settings

因此它更适合进入 KB 作为“离线 retrieval policy optimizer”设计灵感，而不是作为可直接照搬的生产实现。

## 对 agent-os 的借鉴

agent-os 的 memory / context recall 不应该长期依赖手写固定参数。更好的路径是定义一个小型 dev set，离线优化：

- `top_k`
- semantic / keyword / graph fusion weights
- node/evidence recall budget
- query rewrite / decomposition
- rerank / MMR 参数

部署时只发布通过 guard 的 retrieval config，而不是让线上 agent 临场试错。

## 来源

- 项目：`/Users/neo/Desktop/project/git/SimpleMem`
- 版本：`v0.3.0-2-g74174a1`
- 状态：从 shelf 定向提升的单源 insight；SimpleMem 整体仍不 wholesale 合并 wiki
