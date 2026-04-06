---
title: "Multi-Agent Orchestration — Claude Code"
category: L2
parent: "[[multi-agent]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的多智能体系统不是"在 prompt 里允许开子会话"，而是建立在正式任务系统之上的 agent orchestration runtime。每个子 agent 拥有独立的执行环境、专属的 MCP servers、独立的 transcript，并通过 `AgentTool` 作为统一入口被调度。Coordinator 模式提供了明确的层级结构，将 orchestrator 与 worker 的职责彻底分离。

## 架构分析

### AgentTool — 多智能体的正式入口

`AgentTool` 是子 agent 的唯一标准化入口点，它不是一个临时机制，而是与 BashTool、FileReadTool 同级的正式 tool。其职责包括：

- 指定 agent type、model 和工作目录（cwd isolation）
- 决定前台（blocking）或后台（task-based）运行模式
- 接入任务系统，使 agent 运行结果可持久化、可恢复
- 在需要时路由至 remote agent / worktree / teammate 模式

这一设计的关键含义：子 agent 的调度是类型化的，而不是任意的自然语言委托。

### runAgent.ts — 子 Agent 的独立执行环境

`runAgent.ts` 是真正执行子 agent 推理回路的核心。它的设计决策揭示了 Claude Code 的隔离哲学：

- **独立 prompt 构建**：子 agent 有自己的 `promptMessages`，不是共享父 agent 的上下文窗口
- **选择性上下文继承**：可继承或扩展父上下文（`createSubagentContext`），但隔离是默认行为
- **专属 MCP servers**：子 agent 可在 frontmatter 中声明自己的 MCP server 列表，在启动时初始化（`initializeAgentMcpServers`），运行结束后清理
- **独立 transcript 记录**：通过 `setAgentTranscriptSubdir` / `recordSidechainTranscript` 记录子 agent 专属的对话历史
- **SubagentStart hook 触发**：通过 `executeSubagentStartHooks` 向外暴露生命周期事件

### spawnMultiAgent.ts — 进程与终端编排

多智能体不仅是逻辑问题，还是进程拓扑问题。`spawnMultiAgent.ts` 处理 teammate / worker 的物理启动方式：

- **CLI flags 继承**：子进程继承父进程的标志和权限模式
- **权限传播**：通过 `getSessionBypassPermissionsMode` 保持权限一致性
- **tmux session 管理**：通过 `SWARM_SESSION_NAME` / `TMUX_COMMAND` 在 tmux pane 中启动 worker
- **in-process teammate**：`startInProcessTeammate` / `spawnInProcessTeammate` 支持同进程内运行 teammate，避免进程开销
- **swarm backend 选择**：`detectAndGetBackend` 自动选择 tmux、in-process 或其他 backend

这意味着 Claude Code 的 swarm 模式有三种拓扑：同进程、同机器多进程（tmux）、以及未来可能的远程进程。

### coordinatorMode.ts — 明确的组织层级

`coordinatorMode.ts` 将 coordinator 建模为独立的运行模式（通过 `CLAUDE_CODE_COORDINATOR_MODE` 环境变量激活），提供：

- **Coordinator 专用系统提示**：与普通 agent 不同的 system prompt，明确说明 coordinator 的职责（研究分发、结果综合、验证标准、并发调度）
- **Worker capability 声明**：向 coordinator 描述 worker 的能力和任务协议
- **工具集限制**：coordinator 和 worker 各有不同的允许工具集（`ASYNC_AGENT_ALLOWED_TOOLS` / `INTERNAL_WORKER_TOOLS`）
- **模式恢复**：`matchSessionMode` 在 resume 时检查会话模式，必要时切换环境变量

### 任务系统 — 状态持久化的基础

子 agent 的运行结果被包装成 task，赋予了以下能力：
- **可后台运行**：agent 在后台执行，主线程不阻塞
- **可恢复**：任务状态可持久化到磁盘，支持 `compact` / `resume`
- **可继续发消息**：通过 `writeToMailbox` 向运行中的 agent 发送消息
- **可前台接管**：后台 agent 可被提升为前台会话
- **有结构化输出**：transcript 和 metadata 分离存储

### 关键代码路径

- `src/tools/AgentTool/AgentTool.tsx` — 多智能体入口，参数定义与模式路由
- `src/tools/AgentTool/runAgent.ts` — 子 agent 执行回路，MCP 初始化，transcript 记录
- `src/tools/shared/spawnMultiAgent.ts` — Teammate 进程/终端编排，backend 选择
- `src/coordinator/coordinatorMode.ts` — Coordinator 模式系统提示，工具集定义，模式恢复
- `src/utils/swarm/spawnInProcess.ts` — 同进程 teammate 启动
- `src/utils/swarm/backends/registry.ts` — Swarm backend 检测与注册

## 设计亮点

- **任务系统先于多智能体**：子 agent 结果被包装为 task，而不是临时子会话，这使得 async agent、恢复、状态查询成为可能，是架构成熟度的标志
- **执行环境隔离**：子 agent 有独立的 MCP 连接、独立 transcript、独立 abort controller，防止跨 agent 状态污染
- **拓扑弹性**：同一套接口（`spawnMultiAgent`）可以透明地路由到 in-process、tmux、或远程 backend，上层 agent 无需关心物理部署方式
- **Coordinator 作为一等公民**：通过专用 system prompt 和工具集约束，coordinator 的角色被明确编码到运行时，而不是依赖 LLM 自行理解"我是 coordinator"

## 局限性

- **Coordinator 模式目前是单机的**：`CLAUDE_CODE_COORDINATOR_MODE` 环境变量意味着 coordinator 和 worker 必须共享同一套配置，真正的分布式协调尚未实现
- **Agent 间通信通过 mailbox 而非共享内存**：`writeToMailbox` 是异步写文件的机制，延迟较高，不适合细粒度的实时协作
- **Worker 工具集在代码中硬编码**：`INTERNAL_WORKER_TOOLS` 和 `ASYNC_AGENT_ALLOWED_TOOLS` 是静态常量，动态工具能力协商尚不支持
- **in-process backend 仍是实验性的**：`markInProcessFallback` 存在说明 in-process 模式有已知的回退场景

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
