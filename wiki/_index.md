---
title: AI Agent Knowledge Base
category: index
updated: 2026-04-25
---

# AI Agent Knowledge Base

聚焦智能体开发的结构化知识库。以组件为维度，跨项目对比设计。

## 概念地图

### Core Runtime
- [[prompt-system]] — 动态组装 prompt
- [[query-loop]] — Agent 主循环
- [[runtime-state]] — 运行时状态与生命周期
- [[context-management]] — Context window 管理

### Capabilities
- [[tool-system]] — 工具发现/调用/管理
- [[memory-system]] — 跨会话持久记忆
- [[multi-agent]] — 多 agent 协作
- [[agent-registry-discovery]] — Agent 注册与发现（Agent OS 的地址总线）
- [[finetuning-system]] — Agent/LLM 微调与训练管道

### Extension
- [[hooks]] — 事件驱动扩展点
- [[mcp-skills]] — 外部扩展协议 + 能力包
- [[channel-remote]] — 多渠道接入

### Reliability
- [[session-recovery]] — 断点恢复与容错
- [[evaluation-observability]] — 效果评估 + 运行可观测
- [[sandbox-isolation]] — LLM 授权代码执行的隔离机制

### Cross-Concept Patterns
- [[_patterns/_index]] — 跨多个 L1 概念的可迁移架构组合模式

### Single-Source Insights
- [[simplemem--dual-storage]] — SimpleMem 的 SQLite + LanceDB 双存储记忆架构

## 来源项目

| 项目 | 状态 | L2 页数 | 备注 |
|------|------|---------|------|
| AgentScope | 已分析 | 12 | 唯一覆盖完整 finetuning pipeline 的源 |
| Claude Code | 已分析 | 11 | 含 swarm 多 agent 后端 |
| DeerFlow | 已分析 | 10 | middleware pipeline 设计 |
| Hermes Agent | 已分析 | 11 | 含 `_agent_cache` prompt cache 经济学 |
| MiroFish | 已分析 | — | （无 L2，架构文档级） |
| Mempalace | 已分析 | — | （无 L2，架构文档级） |
| neoagent | 已分析 | 10 | 设计密度最高的单一框架源 |
| OpenHarness | 已分析 | 11 | TeamRecord 内存注册 |
| SimpleMem | 已分析 | 10 | 记忆系统专项，双存储架构 |
| OpenAI Agents SDK | 计划中 | — | |
| Dify | 计划中 | — | |
| LangGraph | 计划中 | — | |

## 关系类型

| 关系 | 含义 |
|------|------|
| A 用了 B | 运行时调用 |
| A 给 B 提供数据 | 数据流向 |
| A 和 B 思路相反 | 理念冲突 |
| A 变成了 B | 技术演化 |
| A 和 B 是替代方案 | 同一问题不同解法 |
| A 给 B 加了新能力 | 扩展 |
