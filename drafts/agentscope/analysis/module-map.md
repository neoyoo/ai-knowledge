# 模块 → 概念维度映射：agentscope

**生成时间**: 2026-04-15
**源路径**: /Users/neo/Desktop/project/git/agentscope
**主语言**: Python
**代码规模**: 约 215 个 Python 文件

## 顶层模块列表

| 模块路径 | 主要功能（推断） | 映射概念维度 |
|---------|---------------|------------|
| agent/ | Agent 主体实现 | query-loop |
| pipeline/ | Pipeline 编排（结构化顺序/并行/条件） | query-loop |
| model/ | 模型调用抽象层 | query-loop, prompt-system |
| formatter/ | 消息格式化 | prompt-system |
| memory/ | 记忆存储与检索 | memory-system |
| rag/ | 检索增强生成 | memory-system, context-management |
| embedding/ | 向量嵌入 | memory-system |
| token/ | Token 计数与预算管理 | context-management |
| message/ | 消息传递与状态 | runtime-state |
| session/ | 会话生命周期管理 | runtime-state, session-recovery |
| hooks/ | 生命周期钩子 | hooks |
| mcp/ | MCP 协议集成 | mcp-skills |
| tool/ | 工具注册与执行 | tool-system |
| a2a/ | Agent-to-Agent 协议 | multi-agent |
| plan/ | 基于计划的任务分发 | multi-agent |
| realtime/ | 实时流式通信 | channel-remote |
| tts/ | 文字转语音通道 | channel-remote |
| tracing/ | 链路追踪与可观测性 | evaluation-observability |
| evaluate/ | 评估框架 | evaluation-observability |
| tune/ | Finetuning 入口 | ⚠️ ontology 未覆盖（unique） |
| tuner/ | Finetuning 具体实现 | ⚠️ ontology 未覆盖（unique） |
| _utils/ | 通用工具函数 | （不单独写 L2） |
| exception/ | 异常定义 | （不单独写 L2） |
| types/ | 类型定义 | （不单独写 L2） |
| module/ | 模块注册系统 | （不单独写 L2） |

## 概念维度覆盖计划

| 概念维度 | 相关模块 | 是否派 subagent |
|---------|---------|---------------|
| query-loop | agent/, pipeline/, model/ | 是 |
| prompt-system | formatter/, model/ | 是 |
| memory-system | memory/, rag/, embedding/ | 是 |
| context-management | token/, rag/ | 是 |
| runtime-state | message/, session/ | 是 |
| session-recovery | session/ | 是 |
| hooks | hooks/ | 是 |
| mcp-skills | mcp/ | 是 |
| tool-system | tool/ | 是 |
| multi-agent | a2a/, plan/ | 是 |
| channel-remote | realtime/, tts/ | 是 |
| evaluation-observability | tracing/, evaluate/ | 是 |

**⚠️ 特别发现**：`tune/` + `tuner/` 模块实现了完整的 finetuning pipeline（SFT/DPO/推理优化），这在现有 ontology 12 个概念维度中无对应。建议 ingest 完成后评估是否升为新 L1 概念。

## 排除模块

| 路径 | 排除原因 |
|------|---------|
| docs/ | 文档 |
| tests/ | 测试代码 |
| examples/ | 示例代码 |
| assets/ | 静态资源 |
| _utils/ | 通用工具，不单独分析 |
| exception/ | 异常定义，无架构意义 |
| types/ | 类型定义 |
| module/ | 模块注册，随具体 L2 提及 |
