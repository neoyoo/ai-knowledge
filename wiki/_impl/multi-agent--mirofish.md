---
title: "Multi-Agent — MiroFish"
category: L2
parent: "[[multi-agent]]"
source: mirofish
source_version: "0.1.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

MiroFish 是 HKUDS/OpenHarness v0.1.0 中的群体智能模拟引擎，用数百个 LLM agent 模拟社交媒体上的信息传播和舆论演化。其核心创新是**环境介导通信（Environment-Mediated Communication）**——agent 之间不直接对话，而是通过共享的社交环境（OASIS，模拟 Twitter/Reddit）间接交互，行动本身即通信。

## 架构分析

### 三层通信架构

```
┌─────────────────────────────────────────────┐
│  Layer 3: 分析提取层                          │
│  ReportAgent + InsightForge / PanoramaSearch  │
│  / QuickSearch tools（事后分析）              │
├─────────────────────────────────────────────┤
│  Layer 2: 记忆积累层                          │
│  Zep 时序知识图谱（批量更新，5条/批）          │
├─────────────────────────────────────────────┤
│  Layer 1: 实时交互层                          │
│  OASIS SQLite（posts / comments /             │
│  follows / reposts / traces 表）              │
└─────────────────────────────────────────────┘
```

**Layer 1 — 实时交互**：Agent 的每次社交行为（发帖、回复、点赞、转发）即时写入 SQLite。这是 agent 间信息流动的物理基础。

**Layer 2 — 记忆积累**：Zep 时序知识图谱随模拟进程因果演化。不是快照，而是带时间戳的事件链。批量写入（5条/批）平衡了实时性与开销。

**Layer 3 — 分析提取**：模拟结束后，ReportAgent 通过 ReACT 循环调用 InsightForge（深度分析）、PanoramaSearch（全局视图）、QuickSearch（快速检索）从演化后的图谱中提取洞察。

### 通信核心机制：OASIS 推荐算法

```
Round N:   Agent A ──发帖──→ SQLite posts 表
                                    ↓
                         OASIS 推荐算法（amplification）
                                    ↓
Round N+1: Agent B ←──推送到 timeline── 
```

Agent 之间的影响是**严格异步、轮次间延迟**的。共识不是协商出来的，而是通过推荐算法的放大效应自然涌现：热门观点扩散，冷门观点衰减。

### 关键代码路径

**`backend/scripts/run_parallel_simulation.py`**
主循环入口。调用 `OASIS env.step()` 推进每一轮，记录 action 日志。并行运行 Twitter + Reddit 双平台。

**`backend/app/services/zep_graph_memory_updater.py`**
实时监听 SQLite 变更，批量（5条/批）写入 Zep 知识图谱。是 Layer 1 → Layer 2 的桥接层。

**`backend/app/services/simulation_ipc.py`**
基于文件的 IPC 接口，允许外部向运行中的模拟注入事件（如突发新闻）。解耦了外部刺激与内部 agent 循环。

**`backend/app/services/report_agent.py`**
模拟结束后启动，以 ReACT 模式从知识图谱生成分析报告。体现了"模拟结束，世界仍在"的设计理念。

## 设计亮点

### "行动即通信"（Action IS Communication）

这是 MiroFish 最核心的设计哲学。传统多 agent 系统需要显式的消息传递 API；MiroFish 中 agent 的每次社交行为（发帖/回复/转发）本身就是通信的载体。没有额外的通信层，通信成本为零，但表达能力更丰富——一个转发行为携带的信息（认同、传播、受众扩展）远比一条消息更多。

### 去中心化共识涌现

没有 Coordinator agent 来判断是否达成共识。共识是通过推荐算法的自然选择机制涌现的：

- 被广泛转发的观点 → 更高曝光 → 更多认同 → 形成主流
- 无人响应的观点 → 自然衰减 → 消失

这与现实社交媒体中的信息传播机制完全同构，使得模拟结果具有更高的生态效度。

### 时序知识图谱的因果链

Zep 图谱不是状态快照，而是带时间戳的因果事件链。这意味着可以追溯：观点 X 在第几轮被 Agent A 提出 → 第几轮被 Agent B 转发并变异 → 最终在哪一轮形成共识。模拟结束后，图谱保留完整的演化路径，支持后续深度分析。

### 模拟后世界仍存活

`report_agent.py` 可以在模拟结束后继续"采访"agent，因为 Zep 图谱中保留了 agent 的完整记忆状态。这突破了传统模拟"结束即清零"的限制，支持反事实实验（"如果在第 10 轮注入这条新闻，结果会怎样？"）。

### 双平台并行模拟

Twitter + Reddit 同时运行，同一批 agent 可以在两个平台上呈现不同的行为模式（Twitter 的短促传播 vs Reddit 的深度讨论）。通过比较双平台的共识演化路径，可以研究平台机制对信息传播的影响。

## 局限性

**重基础设施依赖**
OASIS（模拟社交环境）+ SQLite + Zep Cloud 三层栈，部署复杂度高。Zep Cloud 是外部服务依赖，引入网络延迟和成本。

**无实时 P2P 通信**
所有通信都经过 OASIS 环境中介，agent 之间不能直接交换消息。对于需要快速协商的任务（如实时竞标、即时协作），延迟一轮的间接通信不适用。

**共识无法保证收敛**
基于推荐算法的涌现共识是概率性的，不保证一定收敛，也不保证收敛到"正确"答案。对于需要确定性共识的应用场景（如投票、合同谈判），这不是合适的架构。

**固定轮次终止，无动态收敛检测**
模拟以固定轮次终止，不能根据观点分布的稳定性自适应停止。轮次设少了可能截断演化过程，设多了浪费计算资源。

**观点极化风险**
推荐算法的放大效应在现实社交媒体中已被证明会导致信息茧房和观点极化。在模拟场景下，同样可能出现少数极端观点被过度放大的情况，影响模拟结果的代表性。

## 与其他方案的对比定位

| 对比维度 | MiroFish | Claude Code AgentTool | OpenHarness 子进程 |
|----------|----------|----------------------|------------------|
| 通信方式 | 环境介导（异步，轮次延迟） | 直接 task 分发（同步返回） | stdin/stdout 消息行（准同步） |
| 协调机制 | 去中心化涌现 | 中央 Orchestrator | 无（单向消息） |
| 适合规模 | 数百 agent | 十级 agent | 十级 agent |
| 共识机制 | 自然选择放大 | Orchestrator 显式合并 | 无内置共识 |
| 记忆共享 | Zep 图谱（时序） | 独立 transcript（隔离） | 仅日志文件 |

## 适用场景

MiroFish 的架构适合：
- 社会模拟和舆论演化研究
- 大规模 agent 群体行为建模（>100 agent）
- 需要研究信息传播路径和影响机制的场景
- 对实时性要求不高、可接受轮次延迟的协作场景

不适合：
- 需要实时协商和即时反馈的任务
- 需要保证共识确定性的场景
- 轻量级多 agent 协作（引入 OASIS+Zep 代价过高）

## 来源

- **项目**: HKUDS/OpenHarness
- **版本**: v0.1.0
- **分析深度**: 源码级（关键服务模块 + 主循环脚本）
- **置信度**: high
