---
title: "tool-system——agentscope"
category: L2
parent: "[[tool-system]]"
source: agentscope
source_version: "v2.0.1-11-g0e5418e8"
concept: tool-system
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的工具系统以 `ToolBase`、`ToolGroup`、`Toolkit` 和 `AgentState.tool_context` 为核心。工具是对象协议，不再是旧版 `register_tool_function()` 的函数注册模型；MCP 工具、workspace 内置工具、planning tools、team tools、scheduler tools 和 background task tools 都以 `ToolBase` 形态汇入 `Toolkit`。

工具组激活状态保存在 `AgentState.tool_context.activated_groups`，LLM 可通过内置 `ResetTools` 元工具切换可见工具组。工具执行统一产出 `ToolChunk` stream，最终累积为 `ToolResponse` 写回 context。

---

## 架构分析

### 核心对象

| 对象 | 职责 |
|---|---|
| `ToolBase` | 工具协议：name、description、input_schema、permission check、read-only 判断 |
| `ToolGroup` | 一组 tools / skills / MCP clients，可被 agent 动态激活 |
| `Toolkit` | 根据 activated groups 聚合可见工具、执行 tool call、渲染 skill instructions |
| `AgentState.tool_context` | 保存 activated groups 和 read-file cache |
| `ToolChunk` / `ToolResponse` | 流式工具结果与最终工具结果 |

### 工具来源

`ChatService` 每轮通过 `get_toolkit()` 组装完整工具集：

1. Workspace builtins：Bash / Read / Write / Edit / Glob / Grep 等
2. Planning tools：`TaskCreate` / `TaskList` / `TaskGet` / `TaskUpdate`
3. Background task control：如 `TaskStop`
4. Schedule tools：创建、查看、删除、列出 cron schedule
5. Team tools：leader 拿 `TeamCreate` / `AgentCreate` / `TeamSay` / `TeamDelete`，worker 只拿 `TeamSay`
6. Caller-supplied extra tools
7. Workspace skills 与 MCPs

### 可见性与动态切换

`Toolkit.get_tool_schemas(groups)` 只返回：

- 永远可见的 `basic` group
- 当前 `state.tool_context.activated_groups` 中已激活的 group
- 当存在非 basic group 时，内置 `ResetTools` 元工具
- 当存在可用 skill 时，内置 `SkillViewer`

这让工具集可以在长任务中按需缩放，减少默认工具 schema 膨胀。

### 执行路径

`Agent._execute_tool_call()` 负责权限、事件、context 写入；`Toolkit.call_tool()` 只负责调用工具并把返回结果规范化为 stream：

1. 根据 activated groups 找工具
2. 解析 tool call JSON input
3. 对非 MCP / 非 external tool 注入 `_agent_state`（若工具声明 `is_state_injected`）
4. 调用工具，支持 coroutine、async generator、sync generator、单个 `ToolChunk`
5. 捕获 MCP error 和普通异常，转成 error `ToolChunk`
6. 最后 yield 完整 `ToolResponse`

### 权限模型

`ToolBase` 提供三层权限支持：

- `check_read_only(tool_input)`：调用级只读判断
- `check_permissions(tool_input, PermissionContext)`：工具自定义权限判断
- `match_rule()` / `generate_suggestions()`：权限规则匹配与建议

MCP 工具会读取 MCP annotation 的 `readOnlyHint`，只读工具默认 allow，非只读默认 ask。

---

## 关键代码路径

- `src/agentscope/tool/_base.py` — `ToolBase` 权限协议
- `src/agentscope/tool/_toolkit.py` — `Toolkit.get_tool_schemas()` / `call_tool()`
- `src/agentscope/tool/_tool_group.py` — `ToolGroup`
- `src/agentscope/tool/_adapters.py` — `FunctionTool` / `MCPTool`
- `src/agentscope/tool/_builtin/_meta.py` — `ResetTools`
- `src/agentscope/tool/_builtin/_skill.py` — `SkillViewer`
- `src/agentscope/state/_state.py` — `ToolContext.activated_groups`
- `src/agentscope/app/_service/_toolkit.py` — 每轮工具集装配
- `src/agentscope/agent/_agent.py` — `_execute_tool_call()` / `_acting()`

---

## 设计亮点

### 1. ToolBase 把权限作为工具协议的一部分

工具不是裸函数，而是带 permission check、read-only 判断、规则匹配和建议生成的对象。对 Web/distributed agent 来说，这比“函数 + schema”更适合接入审批 UI 和租户策略。

### 2. ToolGroup 是工具、MCP、Skill 的统一可见性单位

同一个 group 可以同时包含工具、MCP clients 和 skills。LLM 激活一个 group 后，不只获得工具 schema，也能获得对应 skill instructions。

### 3. LLM 可通过 ResetTools 自主管理工具集

`ResetTools` 的 schema 根据当前 tool groups 动态生成。agent 可先在 basic 工具集下思考，再按任务需要激活 schedule/team 等工具组，降低默认 prompt/schema 压力。

### 4. 工具执行和上下文写入分离

`Toolkit.call_tool()` 不直接写 agent context；context 写入在 `Agent._execute_tool_call()` 中完成。这使 `on_acting` middleware 可以安全包裹纯 I/O，不必承担 state mutation。

### 5. App 级工具装配统一

HTTP chat、wakeup、team worker 都通过 `ChatService` 和 `get_toolkit()` 走同一条装配路径。工具来源一致，减少入口分叉。

---

## 局限性

### 1. Toolkit 本身不持久化

工具组激活状态在 `AgentState.tool_context` 中；工具对象每轮由 workspace/storage/app managers 重新装配。恢复 session 时必须保证工具注册来源一致，否则旧 context 中的工具名可能无法解析。

### 2. 动态工具组依赖模型正确调用 ResetTools

若模型不知道需要激活某组工具，会出现“工具存在但不可见”的失败。重要工具组 description 必须写得足够任务导向，必要时由 policy/middleware 自动激活。

### 3. Skill 是软约束

SkillViewer 让模型读取 skill，但 skill 执行质量依赖模型遵循说明。关键能力仍应实现为 `ToolBase` 或 MCP。

### 4. MCP 工具权限粒度有限

`MCPTool.match_rule()` 没有针对 MCP 参数做细粒度匹配，默认只能 tool-name-level 规则。高风险 MCP server 需要额外 wrapper 或 server 侧权限。

### 5. Tool schema 体积仍可能膨胀

ToolGroup 能降低默认可见工具数，但一旦激活大 group，所有工具 schema 仍会进入模型输入。大规模 MCP 生态仍需要类似 lazy discovery / tool search 的二级发现机制。

---

## 对 agent-os 的借鉴

agent-os 的工具系统建议采用四层抽象：

- `ToolBase`：工具协议 + 权限协议 + schema
- `ToolProvider`：workspace、MCP、remote agent、plugin 产出工具
- `ToolGroupPolicy`：决定哪些 group 默认可见、哪些由模型或策略激活
- `ToolExecutionRuntime`：权限、事件、stream、context 写入、background/offload

本地代理可以使用 in-process provider；Web distributed agent 应通过 workspace/session 重新装配 tools，并把 active groups 存在 session state，而不是序列化工具对象。

---

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
- 核心文件：
  - `src/agentscope/tool/_base.py`
  - `src/agentscope/tool/_toolkit.py`
  - `src/agentscope/tool/_tool_group.py`
  - `src/agentscope/tool/_adapters.py`
  - `src/agentscope/tool/_builtin/_meta.py`
  - `src/agentscope/tool/_builtin/_skill.py`
  - `src/agentscope/state/_state.py`
  - `src/agentscope/app/_service/_toolkit.py`
  - `src/agentscope/agent/_agent.py`
