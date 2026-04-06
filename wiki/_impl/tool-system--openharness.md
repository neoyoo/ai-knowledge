---
title: "Tool System — OpenHarness"
category: L2
parent: "[[tool-system]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

OpenHarness 的工具系统以 `BaseTool` 抽象类为核心，每个工具是一个携带 Pydantic 输入模型的类，通过 `to_api_schema()` 自动生成 Anthropic 兼容的 JSON Schema。工具注册表是一个普通的 `dict[str, BaseTool]`，调度路径通过 `_execute_tool_call()` 实现输入验证、权限检查、执行、结果封装的完整流程。MCP 工具通过 `McpToolAdapter` 全自动包装，无需手动定义 Pydantic 模型。

## 架构分析

### BaseTool 结构

每个工具是 `BaseTool` 的子类，必须定义四个要素：`name`（字符串标识）、`description`（供模型读取的自然语言描述）、`input_model`（Pydantic `BaseModel` 子类，描述参数结构）、`async execute()`（异步执行方法）。`to_api_schema()` 调用 `input_model.model_json_schema()` 并按 Anthropic API 格式组装，避免手写重复的 JSON Schema。`is_read_only(arguments)` 方法由工具自我声明只读性，用于权限层判断。

### 工具调度路径

`_execute_tool_call()` 是统一入口：先从 `ToolRegistry`（`dict[str, BaseTool]`）查找工具，用 `input_model.model_validate()` 验证并反序列化参数（Pydantic 负责类型检查和错误报告），再执行权限检查，最后调用 `execute()` 并将结果封装为 `ToolResultBlock` 返回。错误路径在验证失败或权限拒绝时提前返回，不进入 `execute()`。

### MCP 工具适配

`McpToolAdapter` 实现 MCP 工具的全自动包装：从 MCP 服务器获取 JSON Schema 后，用 `pydantic.create_model()` 动态生成与 `BaseTool.input_model` 接口兼容的 Pydantic 类。这使 MCP 工具与本地工具在调度层完全同构，无需分支处理。

### 工具注册

`create_default_tool_registry()` 在 `tools/__init__.py` 中统一创建所有内置工具并返回 `dict[str, BaseTool]`。所有工具始终全量暴露给模型，无条件过滤或分组机制。

### 关键代码路径

- `tools/base.py` — `BaseTool` 抽象类，`to_api_schema()`、`is_read_only()`、`_execute_tool_call()` 实现
- `tools/__init__.py` — `create_default_tool_registry()` 工厂函数，内置工具集合
- `tools/mcp_tool.py` — `McpToolAdapter`，基于 `pydantic.create_model()` 的动态模型生成

## 设计亮点

- Pydantic `input_model` 兼顾参数验证与 API Schema 生成，一套定义两用，消除同步负担
- `McpToolAdapter` 通过 `create_model()` 实现 MCP 工具的零手工接入，扩展成本极低
- 工具自我声明只读性（`is_read_only()`），权限逻辑下沉到工具层，调度层无需感知语义
- 注册表用普通 `dict` 而非复杂容器，查找路径完全透明，易于调试

## 局限性

- 无工具分组或上下文条件过滤——所有工具始终全量暴露，在工具数量增多后会增大模型上下文压力
- 无流式工具输入累积（streaming tool input accumulation），长参数场景存在缺口
- `is_read_only()` 仅做布尔分类，无法区分参数级别的读写差异（如同一工具部分参数写入）
- 无工具调用重试或降级策略，失败处理完全由调用方负责

## 来源

- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
