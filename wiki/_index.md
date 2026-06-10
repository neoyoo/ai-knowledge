---
title: AI Agent Knowledge Base
category: index
updated: 2026-06-10
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
- [[simplemem--evolvemem-retrieval-optimizer]] — SimpleMem/EvolveMem 的离线检索策略优化器
- [[tencentdb-agent-memory--mmd-symbolic-context-map]] — TencentDB Agent Memory 的 MMD 高密度符号上下文图

## 来源项目

| 项目 | 状态 | L2 页数 | 备注 |
|------|------|---------|------|
| AgentScope | 已分析 | 14 | Python 2.x 主线：session/workspace/message-bus/team runtime |
| AgentScope Java | 已分析 | 1 | AgentCard / A2A / Nacos / Spring Boot 服务化链路 |
| Claude Code | 已分析 | 12 | 含 swarm 多 agent 后端 |
| DeerFlow | 已分析 | 12 | middleware pipeline 设计 |
| Hermes Agent | 已分析 | 12 | 含 `_agent_cache` prompt cache 经济学 |
| MiroFish | 已分析 | 1 | multi-agent 应用型案例 |
| Mempalace | 已分析 | 1 | backend/source adapter contract 与 raw memory 历史 |
| neoagent | 已分析 | 10 | 设计密度最高的单一框架源 |
| OpenHarness | 已分析 | 12 | TeamRecord 内存注册 |
| SimpleMem | shelf + insight | 10 | 记忆系统专项；EvolveMem 检索优化 insight 已提升 |
| TencentDB Agent Memory | 已分析 | 1 | 记忆系统专项：短期 context offload + 长期 memory + MMD 符号图 |
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
