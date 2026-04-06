---
title: "Tool System — Claude Code"
category: L2
parent: "[[tool-system]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的工具系统不是把模型接到裸函数上，而是构建了一层完整的工具执行运行时（Tool Execution Runtime）。这套运行时由三部分构成：工具协议对象（Tool Protocol Object）、分层工具池（Layered Tool Pool）、以及两套并行的执行引擎（批处理 + 流式）。与此同时，权限控制被深度嵌入工具系统，形成从"模型可见性"到"OS 级沙箱"的双层防线，使工具的安全语义成为运行时的一等成员。

## 架构分析

### 工具抽象：协议对象而非裸函数

`src/Tool.ts` 定义了工具的最小接口规范。每个工具不仅携带执行逻辑，还内建了：

- **Input/Output schema**：用于模型 function calling 的结构化描述
- **`isConcurrencySafe()`**：声明该工具是否支持并发执行（fail-closed 默认策略：未明确声明则视为不安全）
- **`interruptBehavior()`**：声明用户中断时的响应策略，`cancel`（立即中止）或 `block`（等待完成）
- **权限检查入口**：工具自带与权限系统对接的钩子
- **渲染语义**：progress / result / error 的 UI 呈现逻辑

这一设计使工具成为运行时中的"一等实体"，而非单纯的函数回调。

### 分层工具池

`src/tools.ts` 把工具池分为三个逻辑层次：

1. **`getAllBaseTools()`**：代码中定义的全部内建工具（理论上界）
2. **`getTools()`**：当前运行模式下暴露给模型的工具子集
3. **`assembleToolPool()`**：最终工具池，合并内建工具与 MCP 工具

这一分层的核心洞察是：**工具存在 ≠ 工具可见 ≠ 工具可执行**。代码里存在的工具不一定暴露给模型；暴露给模型的工具，在执行前仍需过权限审批。

### ToolUseContext：工具执行的共享上下文

工具执行不是孤立的函数调用，而是在一个富含状态的上下文中运行。`ToolUseContext` 承载了：

- 当前可用工具列表、MCP 客户端
- AppState 的读写接口
- Abort Controller（取消控制）
- 文件缓存
- 通知与 UI 回调
- 当前消息与 agent 身份标识

这意味着工具可以感知并修改 agent 的全局状态，而不仅仅是返回结果。

### 双引擎执行：批处理 + 流式

Claude Code 维护了两套工具执行引擎，分别服务不同场景：

**批处理引擎**（`src/services/tools/toolOrchestration.ts`）：
- 将 tool_use block 划分为执行批次
- 对声明了 `isConcurrencySafe` 的工具并发执行
- 对不安全工具串行执行
- 收集并回放上下文修改（保证幂等）

**流式执行引擎**（`src/services/tools/StreamingToolExecutor.ts`）：
- 在模型流式输出期间提前启动工具（减少等待）
- 管理 progress 状态的实时推送
- 保证结果按接收顺序（而非完成顺序）回传给模型
- 处理中断、并发冲突、fallback discard 等边缘情况

### 权限系统：双层防线

工具系统与权限系统紧密耦合，构成两层防线：

**第一层：工具可见性控制**
`useCanUseTool()` 是权限审批的主入口，支持三种审批路径：
- `interactive`：弹窗向用户确认
- `coordinator`：在 multi-agent 协调器中审批
- `swarm worker`：在 swarm 模式下审批

权限模式由 `src/utils/permissions/PermissionMode.ts` 枚举管理，包括 `default` / `plan` / `acceptEdits` / `bypassPermissions` / `auto` 等策略。自动审批不是跳过检查，而是接入了分类器：低风险命令自动放行，高风险命令仍走确认流程。

**第二层：OS 级沙箱**
`src/utils/sandbox/sandbox-adapter.ts` 把权限配置转换为 sandbox runtime 的执行策略，涵盖：
- 网络域名 allow/deny 规则
- 文件系统读写范围限制
- settings 文件保护（防止 agent 通过修改配置自我提权）
- `.claude/skills` 目录写保护（技能目录视为高权限扩展面）
- git bare repo 逃逸防御（针对真实攻击面的专项补丁）

### 关键代码路径

- `src/Tool.ts` — 工具协议对象的接口定义（并发安全、中断行为、权限入口）
- `src/tools.ts` — 三层工具池（getAllBaseTools / getTools / assembleToolPool）
- `src/services/tools/toolOrchestration.ts` — 批处理执行引擎（并发/串行调度、上下文回放）
- `src/services/tools/StreamingToolExecutor.ts` — 流式执行引擎（提前启动、顺序保证、中断处理）
- `src/hooks/useCanUseTool.tsx` — 权限审批主入口（三路径分发）
- `src/utils/permissions/PermissionMode.ts` — 权限模式枚举与运行时政策
- `src/utils/sandbox/sandbox-adapter.ts` — OS 沙箱策略适配层

## 设计亮点

- **Fail-closed 并发安全**：工具并发需显式声明 `isConcurrencySafe`，未声明默认串行，从根本上避免并发污染
- **流式预启动**：StreamingToolExecutor 在模型还在生成时就提前启动工具，显著降低端到端延迟
- **工具存在与执行解耦**：三层工具池设计使"能力边界"在多个层次独立控制，互不干扰
- **安全是运行时一等成员**：权限审批与 OS 沙箱形成闭环，而非事后加装的附件；settings 文件保护和 skill 目录保护体现了对真实攻击面的预判

## 局限性

- **MCP 工具与内建工具的异构性**：两者在 `assembleToolPool` 中合并，但 MCP 工具的 schema 验证和并发安全声明依赖外部 server 提供，可靠性低于内建工具
- **流式执行的顺序保证代价**：强制按接收顺序回传结果，在高并发场景下可能引入额外等待；fast-first 场景牺牲了吞吐
- **自动审批分类器不透明**：speculative classifier 和 Bash allow classifier 的决策逻辑对用户不可见，难以预测哪些命令会被自动放行

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
