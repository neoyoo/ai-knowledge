---
title: "Query Loop — OpenHarness"
category: L2
parent: "[[query-loop]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述
OpenHarness 的 query loop 以 `run_query()` 为核心，实现为一个最多 8 轮（可配置）的 `for` 循环：每轮流式接收模型响应，若模型返回 tool_use 则执行工具（单工具串行，多工具通过 `asyncio.gather()` 并行），将结果追加为 user 消息后继续下一轮；若无工具调用则直接返回，对话结束。

## 架构分析
### 主循环结构
`run_query()` 是 121 行的 `for _ in range(max_turns)` 循环，默认 `max_turns=8`。每轮执行以下步骤：
1. 调用 `api_client` 流式获取模型响应，收集 text delta 和最终 `ApiMessageCompleteEvent`
2. 将 assistant 消息追加到 `QueryEngine` 持有的对话历史
3. 检查 `message.tool_uses`：若为空则 `return`，退出循环
4. 若有一个工具调用则串行执行；若有多个则通过 `asyncio.gather()` 并行执行全部
5. 将所有工具结果打包为一条 user 消息追加到历史，进入下一轮

### 单工具 vs 多工具分支
单工具与多工具的执行路径在代码中显式分支，逻辑清晰可读。多工具并行是 first-class 设计，不依赖外部调度器，直接使用标准库 `asyncio.gather()`。

### 状态管理
`QueryEngine` 持有完整对话历史（message list）和 `CostTracker`（累计 token 费用）。历史在整个 query 生命周期内原地追加，不做压缩或截断。

### 关键代码路径
- `openharness/engine/query.py` — `run_query()` 主循环，121 行，单/多工具分支逻辑
- `openharness/engine/query_engine.py` — `QueryEngine` 类，持有历史与 `CostTracker`
- `openharness/api/client.py` — 流式 API 调用，产出 text delta 和 `ApiMessageCompleteEvent`

## 设计亮点
- 单/多工具执行路径显式分支，代码意图一目了然，无隐式调度抽象
- `asyncio.gather()` 并行执行多工具是 first-class 设计，不增加框架复杂度
- `CostTracker` 内嵌于 `QueryEngine`，费用追踪随对话历史自然流动

## 局限性
- 无流式 tool-input 累积：工具调用的输入参数无法在生成过程中流向 UI，只能等完整消息后处理
- 超过 `max_turns` 直接抛出 `RuntimeError`，无优雅降级或部分结果返回
- 流式响应不可中断：用户无法在模型输出过程中取消当前轮次
- 对话历史不做压缩，长对话可能撑爆 context window

## 来源
- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
