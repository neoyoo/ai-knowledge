---
title: "Tool System — DeerFlow"
category: L2
parent: "[[tool-system]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 的工具系统通过 `get_available_tools()` 从四个来源聚合工具：配置文件定义的工具（bash、web_search 等）、内置工具（present_file、ask_clarification 等）、MCP 扩展工具以及 ACP 跨框架 agent 工具。其最大特色是"延迟工具注册表"（DeferredToolRegistry）——当 MCP 工具数量庞大时，将其隐藏在 `tool_search` 工具后面，按需发现，从根本上解决了大规模工具列表导致的 token 膨胀问题。安全层面由独立的 `GuardrailMiddleware` 拦截每次工具调用，与调度逻辑完全解耦。

## 架构分析

### 工具聚合层（get_available_tools）

`get_available_tools()` 依次合并四个来源：

1. **配置工具**：从 `config.yaml` 读取，包括 bash、web_search、web_fetch、read_file、write_file 等；
2. **内置工具**：present_file、ask_clarification、view_image、task、skill_manage，硬编码进框架；
3. **MCP 工具**：从 `extensions_config.json` 加载，数量不确定；
4. **ACP agent 工具**：用于跨框架调用其他 harness 中的 agent。

### 延迟工具注册表（DeferredToolRegistry）

当 `tool_search.enabled=True` 时，MCP 工具不直接暴露给模型，而是注册进 `DeferredToolRegistry`。模型只能看到一个 `tool_search` 工具，通过它按需查询和激活具体 MCP 工具。这将模型实际接收的工具列表压缩到可控范围，解决了 MCP 工具集庞大时的 token 预算问题。

### 护栏中间件（GuardrailMiddleware）

每次工具调用都经过 `GuardrailMiddleware`，它调用 `GuardrailProvider.evaluate()` 返回 allow/deny 决定及原因。`is_host_bash_allowed()` 控制 bash 工具是否在宿主机直接执行还是走沙箱隔离。护栏作为独立中间件存在，不耦合进工具调度路径。

### 关键代码路径

- `deerflow/tools/tools.py` — `get_available_tools()` 四源聚合入口
- `deerflow/guardrails/builtin.py` — `GuardrailProvider.evaluate()` 内置护栏实现
- `deerflow/tools/builtins/task_tool.py` — task 内置工具，支持子任务派发
- `deerflow/agents/middlewares/` — GuardrailMiddleware 注册点

## 设计亮点

- **延迟工具注册表**：MCP 工具不进入模型 token 视野，彻底解决大规模工具集的 token 膨胀，是目前所有对比项目中最优雅的解法；
- **护栏独立于调度**：GuardrailMiddleware 是正交的安全层，可独立替换/扩展，不影响工具注册或调度逻辑；
- **ACP 跨框架调用**：通过 ACP 协议将其他框架中的 agent 注册为本地工具，实现跨 harness 组合，扩展性强。

## 局限性

- **护栏能力基础**：内置 GuardrailProvider 为规则型，不支持 LLM 评估的语义级策略（OpenHarness 的 prompt hook 可做到）；
- **tool_search 增加 RTT**：延迟发现需要额外一轮工具调用才能激活 MCP 工具，对低延迟场景有影响；
- **配置分散**：工具来源跨 config.yaml 和 extensions_config.json 两个文件，运维时容易遗漏同步。

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
