---
title: "Runtime State — Claude Code"
category: L2
parent: "[[runtime-state]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的运行时状态系统由 `AppStateStore.ts`（统一状态树）和 `REPL.tsx`（交互式运行时编排器）构成。`AppStateStore` 管理的不是普通 UI 状态，而是整个 agent 工作台的持续运行状态——从工具权限、任务列表、agent 注册表，到 MCP 连接、插件、远程 session；`REPL.tsx` 则不是薄薄的终端渲染层，而是把 prompt 构建、query 驱动、工具审批 UI、任务管理、hooks 和远程连接全部串联起来的前台运行时协调器。两者的关系是"持久状态树 + 交互执行核心"，而非传统的"数据层 + 视图层"。

## 架构分析

### AppStateStore — 工作台状态树

`src/state/AppStateStore.ts` 是整个 Claude Code 前台运行时的共享状态中心。它管理的领域远超 UI 状态的范畴：

**配置与模型层：**
- `settings` — 用户配置，包括模型选择、权限模式、输出风格等
- `model` — 当前使用的底层模型，支持运行时切换

**工具与权限层：**
- `toolPermissionContext` — 工具权限上下文，包含已批准/已拒绝的工具调用记录
- 权限反馈与 permission denial 历史的持久引用

**任务与 Agent 层：**
- `tasks` — 后台任务列表，包含状态（pending / running / done / failed）
- `agentRegistry` — 运行中的 subagent 注册表，支持 coordinator 模式下的多 agent 管理

**MCP 与扩展层：**
- `mcpClients` — 已连接的 MCP server 客户端实例
- `mcpTools` — 从 MCP server 注册的工具集合
- `mcpResources` — MCP server 暴露的资源列表
- `plugins` — 已加载的插件列表

**交互与通知层：**
- `notifications` — 系统通知队列，包括 agent 任务完成通知
- `elicitation` — 待处理的用户确认请求（工具审批、危险操作确认等）

**文件与历史层：**
- `fileHistory` — 当前 session 读写过的文件历史
- `attribution` — 内容来源追踪（用于区分用户输入 vs 模型输出 vs 工具输出）
- `todos` — 任务清单，跨 session 持久化

**远程连接层：**
- `remoteSession` — 远程 agent session 的连接状态
- `bridge` — IDE 集成 bridge 连接（VS Code、JetBrains 等）

这棵状态树的体量说明 Claude Code 把 agent 工作台视为"一个持续运行的工作空间"，而不是"一次请求-响应交互"。

### REPL.tsx — 交互运行时协调器

`src/screens/REPL.tsx` 的职责远超"渲染消息 + 接受输入"：

**Prompt 构建职责：**

REPL 在每次提交前主动调用：
- `getSystemPrompt(...)` — 获取当前模式下的完整 system prompt
- `buildEffectiveSystemPrompt(...)` — 按优先级装配最终 prompt
- `getUserContext()` — 构建用户上下文（文件引用、工作目录等）
- `getSystemContext()` — 构建系统上下文（环境信息、MCP 状态等）

这意味着 REPL 不是 prompt 的消费者，而是 prompt 构建的参与者。每次 query 发出前，REPL 负责把运行时状态转化为 LLM 可消费的上下文。

**Query 驱动职责：**

REPL 直接调用 `query()` 驱动推理主循环，并处理：
- 用户输入的格式化（文本、文件引用、命令前缀处理）
- 流式输出的渐进渲染
- Abort 信号的传递（用户 Ctrl+C → abortController）
- 多轮对话的消息历史维护

**工具审批 UI：**

工具权限审批不是在 query loop 内部静默处理的，而是通过 REPL 展示审批 UI：
- 危险工具调用触发内联确认对话框
- 用户批准/拒绝的结果写入 `AppStateStore.toolPermissionContext`
- 拒绝记录被 query loop 感知，触发 prompt 中"不要原样重试"的行为规则

**任务管理职责：**

REPL 负责显示并管理后台任务列表：
- 展示 coordinator 模式下并发运行的 subagent 状态
- 任务完成通知的接收与展示
- 任务中断与清理

**Compact 后恢复职责：**

当自动压缩发生后，REPL 负责：
- 清理已被压缩的消息显示
- 触发 resume 恢复流程
- 向用户展示压缩提示

**远程与集成职责：**

- SSH 远程连接的建立与状态管理
- IDE bridge 的保活与消息路由
- Voice 输入的接入
- Hooks 系统的生命周期管理（start hook、stop hook、pre/post hook 触发）

### REPL 与 query.ts 的协作关系

两者不是"REPL 包一层 query"的简单嵌套，而是职责明确的分层协作：

```
REPL.tsx                    query.ts
─────────────────────────   ──────────────────────────
构建 prompt / context     →  接收 prompt + messages
处理用户输入格式化         →  执行 queryLoop
渲染流式输出              ←  yield streaming events
展示工具审批 UI           ←→  工具执行前的权限检查
管理任务状态显示          ←  task notification events
处理 compact 后 UI 清理   ←  compact 触发信号
```

`query.ts` 是认知执行核心（LLM + 工具 + 状态机），`REPL.tsx` 是交互执行核心（用户 + UI + 运行时协调）。两者通过 async generator 的 yield 机制解耦，REPL 消费 query 的事件流并相应地更新 UI 和状态。

### AppStateStore 与 REPL 的状态同步

`AppStateStore` 采用响应式状态设计，REPL 和其他组件订阅感兴趣的状态切片：

- 工具权限变更 → 实时更新工具调用按钮状态
- MCP 连接状态变更 → 实时更新可用工具列表
- 任务状态变更 → 实时更新任务面板
- Agent 注册变更 → coordinator 模式下的 subagent 状态追踪

这种设计让跨组件的状态同步不需要手动传递 props，而是通过统一状态树的订阅机制自动传播。

### 关键代码路径

- `src/state/AppStateStore.ts` — 统一状态树，agent 工作台全局状态
- `src/screens/REPL.tsx` — 交互式运行时协调器，prompt 构建，query 驱动
- `src/query.ts` — 推理主循环，与 REPL 通过 async generator 解耦
- `src/utils/systemPrompt.ts` — prompt 装配，REPL 在每次提交前调用
- `src/hooks/useCanUseTool.tsx` — 工具权限检查，REPL 工具审批 UI 的权限依据
- `src/utils/permissions/PermissionMode.ts` — 权限模式定义，绑定到 AppStateStore.settings
- `src/services/compact/autoCompact.ts` — 自动压缩，触发 REPL 的 compact 后清理流程

## 设计亮点

- **统一状态树承载完整工作台语义**：`AppStateStore` 的设计远超 UI 框架通常的状态管理范畴——它同时承载了 agent 的权限上下文、任务注册表、MCP 连接和 IDE 集成状态。这种"把整个 agent 工作台视为一个工作空间"的设计，让跨组件的协调成本极低。
- **REPL 作为运行时编排器而非薄渲染层**：把 prompt 构建、query 驱动、工具审批、任务管理、远程连接全部集中在 REPL 中处理，保持了职责边界清晰——REPL 是"交互域的协调者"，query 是"认知域的执行者"，两者通过 async generator 解耦。
- **权限审批的 UI 闭环**：工具审批 UI 与权限记录（toolPermissionContext）和 prompt 规则（拒绝后不重试）形成完整闭环，用户的每次审批行为都真实反映为模型的下一步行为约束，不是形式化操作。
- **Compact 后恢复的透明处理**：压缩触发和 UI 清理都在 REPL 层处理，对 query.ts 的状态机逻辑透明，体现了关注点分离——核心循环不需要知道 UI 如何处理压缩事件。

## 局限性

- **AppStateStore 体量庞大，单一全局状态树的边界不清**：随着功能增加，AppStateStore 覆盖的领域越来越多（MCP、插件、远程、todos、bridge……），状态树的粒度和边界没有明确规范，新功能容易被随意挂载，造成状态树膨胀。
- **REPL.tsx 职责过重**：prompt 构建、渲染、工具审批、任务管理、远程连接等职责全部在同一个文件中处理，文件体量和认知复杂度都较高，局部修改的影响面难以评估。
- **远程 session 与本地 REPL 的行为一致性缺乏保障**：远程连接（SSH、bridge）的状态管理集成在 REPL 中，但两者在网络中断、重连、状态同步等场景下的一致性没有专门的抽象层保障。
- **任务列表与 agent 注册表缺乏生命周期追踪**：AppStateStore 持有 tasks 和 agentRegistry，但没有显式的任务生命周期状态机（如 timeout、partial failure 的处理策略），依赖上层逻辑处理，容易产生僵尸任务。

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
