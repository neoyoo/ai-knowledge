---
title: Multi-Agent
aliases: [多智能体, multi-agent orchestration, task delegation]
category: L1
created: 2026-04-06
updated: 2026-04-15
relations:
  - target: "[[query-loop]]"
    type: uses
  - target: "[[runtime-state]]"
    type: uses
  - target: "[[memory-system]]"
    type: depends_on
    note: "MsgHub 广播依赖 AgentBase 的 observe() 将消息写入 memory；PlanNotebook 的计划状态本质是结构化 working memory"
  - target: "[[tool-system]]"
    type: depends_on
    note: "PlanNotebook 将计划管理能力封装为 tool set（StateModule），agent 通过 tool call 驱动子任务状态机；这是'工具化状态机'模式的典型实现"
sources: [claude-code, openharness, mirofish, deer-flow, hermes-agent, agentscope]
---

## 一句话定义

多个 agent 协作 — 怎么拆任务、怎么分发、怎么汇总结果。

## 核心问题

- 什么时候该拆成多个 agent，什么时候单个就够？
- Agent 之间怎么通信？共享状态还是消息传递？
- 子 agent 失败了怎么处理？
- 怎么避免重复工作？

## 各家对比

| 维度 | Claude Code | OpenHarness | MiroFish | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|----------|-------------|
| 核心设计 | 建立在正式任务系统之上的 agent orchestration runtime：每个子 agent 拥有独立执行环境、专属 MCP servers 和独立 transcript，通过 `AgentTool` 作为统一标准化入口被调度 | 以操作系统进程为隔离边界：每个子 agent 是独立的 `python -m openharness --headless` 子进程，以 UTF-8 文本行为通信协议；`BackgroundTaskManager` 负责进程生命周期，`SendMessageTool` 向子进程 stdin 写消息；核心逻辑约 280 行 | 环境介导通信——数百 agent 通过共享社交环境间接交互，行动即通信 | `task` 工具将主 agent 升级为 lead agent，子任务交给独立的 SubagentExecutor 实例在线程池中异步并行运行；双线程池（scheduler + execution）混合 async 事件循环；ACP 协议桥接外部 agent（Codex、Claude Code）纳入调度 | `delegate_task` 为统一入口，支持单任务（主线程直接运行）和批量并行（`ThreadPoolExecutor` 最多 3 个子 agent）两种模式；深度上限 `MAX_DEPTH=2`，超限立即拒绝；`frozenset` 白名单在代码层面剥除子 agent 的危险工具；ACP 传输支持委托给异构 agent（Claude CLI、Copilot） |
| 关键特点 | 任务系统先于多智能体（子 agent 结果包装为持久化 task）；拓扑弹性（同进程/tmux 多进程/远程 backend 透明切换）；Coordinator 作为一等公民（专用 system prompt + 工具集约束） | 零依赖隔离（子进程天然隔离，无共享内存、无锁）；极简通信协议（UTF-8 文本行，任何语言可互操作）；broken pipe 检测后自动重启子进程，提升长时任务稳定性 | 去中心化共识涌现；Zep 时序图谱做共享记忆；双平台并行模拟 | 线程池 + async 混合实现真并行；ACP 协议将外部 agent 系统纳入子 agent 调度；stream 事件实时透传子任务进度；SubagentLimitMiddleware 中间件限流（单次 model response 最多 3 个并发 task） | 父子 agent 各有独立 `IterationBudget`（父 90 次，子 50 次，不共享扣减）；所有子 agent 在主线程串行构造后再并发执行（消除全局变量竞态）；`interrupt()` 通过 `_active_children` 线程安全地级联传播到所有深度；`mixture_of_agents_tool` 作为纯推理合成的补充路径（arXiv:2406.04692） |
| 局限 | Coordinator 模式目前是单机的，缺乏真正的分布式协调；agent 间通过 mailbox（异步写文件）通信，延迟较高 | `TeamRecord` 仅存于内存，进程重启后团队关系丢失；单向消息通信，子 agent 无法主动回调协调者；无结果合并机制，子 agent 产出仅写入日志文件 | 重基础设施依赖；无法保证收敛；固定轮次终止 | 无递归嵌套（子 agent 不能再派发子 agent）；无 peer-to-peer 通信；5 秒轮询延迟；子 agent 无持久状态（每次 task 调用创建全新实例） | 父 agent 委托期间同步阻塞（无法处理并发输入）；`MAX_DEPTH=2` 硬编码不可配置；子 agent 无法向父 agent 实时反馈（`clarify`/`send_message` 均被封锁）；批量任务超过 3 个时超出部分直接静默丢弃 |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 单 Agent + 工具 | 一个 agent 调工具完成所有事，不派发子 agent | 大多数任务（90% 的情况不需要多 agent） | 所有基础 agent 框架 |
| 主从派发（Orchestrator + Workers） | 主 agent 把任务分解后派发给子 agent 并行执行，汇总结果 | 可并行的子任务（多文件分析、多维度调研） | Claude Code AgentTool、Hermes Agent delegate_task |
| 对等协作（Peer Agents） | 多个 agent 作为平等节点互相通信，无主从层级 | 多角度审查、辩论、创意生成 | 学术 multi-agent 框架 |
| 流水线（Pipeline） | Agent A 输出送入 Agent B，依次串行处理 | 多阶段工作流（调研 → 写作 → 审查） | OpenHarness 子进程链 |
| 多模型合成（Mixture-of-Agents） | 多个 frontier 模型并行生成答案，再由聚合模型综合 | 纯推理/答案合成，无副作用，追求覆盖单模型盲区 | Hermes Agent mixture_of_agents_tool（arXiv:2406.04692） |

### 场景决策指南

**如果任务能用一个 agent 搞定 → 不要用多 agent**
多 agent 的协调开销、结果合并复杂度、调试难度都会显著上升。先问自己：加几个工具够不够？能够在单 context 里完成的事，不要拆。

**如果需要并行搜索或多维度分析 → 主从派发**
典型场景：同时分析 10 个代码文件、并行调研 5 个竞品。Claude Code 的 `AgentTool` 是这个模式的工业级实现——子 agent 有独立 transcript 和独立 MCP 连接，执行环境互不干扰。Hermes Agent 的 `delegate_task` 批量模式最多并发 3 个子 agent，父 agent 在委托期间同步阻塞，适合无需实时指导的独立子任务。

**如果需要多模型答案合成而非执行操作 → MoA（Mixture-of-Agents）**
当任务是纯推理问题（写作润色、复杂分析、多角度评估），且不需要调用外部工具时，MoA 比 delegate_task 更合适：用 4 个不同 frontier 模型并行生成回答，再由聚合模型综合，能超越单模型天花板且完全无副作用。代价是强依赖 OpenRouter，且无法用于需要工具执行的任务。

**如果需要多角度审查或辩论 → 对等协作，但必须限制轮次**
这个模式容易失控：没有明确终止条件的 peer agent 会反复绕圈。硬限制对话轮次（比如最多 3 轮），并在 prompt 里明确终止条件。

**如果是固定的多阶段工作流 → 流水线**
阶段间职责清晰（调研/写作/审查），每个 agent 专注单一角色。代价是串行延迟叠加，前一阶段的错误会被放大传递到后面。

### 常见陷阱

- **多 agent 是默认选项**：只有当任务明确需要并行、或超过单个 context 上限时才考虑多 agent。复杂度的代价被严重低估。
- **子 agent 之间共享可变状态**：共享状态导致竞态条件和数据不一致。Claude Code 的做法是隔离为默认——独立 transcript、独立 MCP 连接，共享是显式声明的例外。
- **没有限制 agent 间通信轮次**：对等协作或主从模式如果没有轮次上限，轻则浪费大量 token，重则无限循环。每个多 agent 系统都需要一个硬性的 max_turns。
- **子 agent 出错没有回退路径**：流水线模式下，Agent B 收到 Agent A 的错误输出后如果不做验证，错误会被当成正常输入继续传播。每个阶段需要显式的输入校验和失败处理逻辑。
- **结果合并丢失细节**：主从派发在汇总子 agent 输出时，LLM 的摘要会丢失关键细节。需要明确设计结果合并策略，而不是让 orchestrator 随意总结。
- **子 agent 拿到不该有的工具**：如果不在代码层面硬性剔除，子 agent 可以调用 `clarify`（打断用户）、写入共享 `memory`（污染全局状态）、递归再派发（深度爆炸）。Hermes 用 `frozenset(DELEGATE_BLOCKED_TOOLS)` 在工具函数名和工具集名两个层面双重拦截——这是比 system prompt 软约束更可靠的机制。
- **在子线程里构造子 agent 引发竞态**：子 agent 构造过程中通常会修改进程级全局变量（如已解析的工具名列表）。如果多个子 agent 在 ThreadPoolExecutor 的工作线程里并发构造，这些全局变量会相互覆盖。正确做法是在主线程串行完成所有子 agent 构造，再提交给线程池并发执行（Hermes 的实践）。
- **中断信号传不到子 agent**：用户按下 Ctrl-C 时，如果只给父 agent 设置中断标志，正在运行的子 agent 会继续消耗 token 直到自然结束。需要维护一个线程安全的 `_active_children` 列表，在 `interrupt()` 中递归级联传播。

### 实践验证：共享黑板模式的四角色架构

基于 MiroFish 启发 + 实际 MVP 验证，多 agent 协作讨论需要四个核心角色：

| 角色 | 职责 | 时机 | 为什么不能少 |
|------|------|------|------------|
| **Supervisor（监管者）** | 主题锚定，偏离立即纠正 | 每次发言后 | 仅靠 prompt 约束不够，agent 会跑题 |
| **Compressor（压缩器）** | 压缩历史，保留语义 | 上下文达 60% 阈值时 | 讨论内容增长导致 token 爆炸和注意力衰减 |
| **Coordinator（协调者）** | 共识判断，引导焦点 | 每轮结束后 | 没有协调者讨论会发散不收敛 |
| **Agents（讨论者）** | 实际讨论 | 轮内发言 | 核心参与者 |

**关键教训（来自实测）：**
- 主题锚定必须是实时的（每次发言后检查），不能只在轮次结束时检查——到那时已经跑偏了
- 压缩时机应该按 token 占比触发（如 60%），不是按固定轮次——短讨论不需要压缩，长讨论可能第 2 轮就要压
- 压缩必须保留结构化语义（共识/立场/细节/分歧/风险），不能只做简单摘要

## L2 详情

- [[multi-agent--claude-code]]
- [[multi-agent--openharness]]
- [[multi-agent--mirofish]]
- [[multi-agent--deer-flow]]
- [[multi-agent--hermes-agent]]
