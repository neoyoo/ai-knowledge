---
title: "prompt-system——agentscope"
category: L2
parent: "[[prompt-system]]"
source: "agentscope"
source_version: "v2.0.1-11-g0e5418e8"
concept: "prompt-system"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的 prompt system 由两部分组成：

1. `Agent._get_system_prompt()`：运行时组装 system prompt，允许 middleware 顺序改写。
2. `FormatterBase` 及各 provider formatter：把 `Msg` / content blocks 转成 OpenAI、Anthropic、Gemini、DashScope、Ollama、DeepSeek、Moonshot、xAI 等 API 所需消息格式。

当前源码中没有旧页描述的 `TruncatedFormatterBase`，formatter 不再持有 `token_counter/max_tokens` 做自动截断；context 压缩转到 `Agent.compress_context()` 和 `model.count_tokens()`。

---

## 架构分析

### System prompt 组装

模型输入由 `Agent._prepare_model_input()` 统一准备，大致包含：

- system prompt：基础 `system_prompt` + workspace/offloader instructions + skill instructions + middleware 改写结果
- `state.summary`：压缩后的历史摘要
- `state.context`：未压缩消息历史
- tools schema：`toolkit.get_tool_schemas(state.tool_context.activated_groups)`

`on_system_prompt` middleware 是顺序 transformer：每个 middleware 接收当前 prompt 字符串并返回新字符串。

### Formatter 层

`FormatterBase` 只定义 provider message formatting 的公共能力：

- `format(list[Msg]) -> list[dict]`
- `assert_list_of_msgs()`
- `_group_messages()`：把消息分成 `tool_sequence` / `agent_message`
- `convert_tool_result_to_string()`：将不被目标 provider 支持的多模态 tool result 转为文本提醒或可提升的 data block

真实 formatter 包括：

- `OpenAIChatFormatter` / `OpenAIMultiAgentFormatter`
- `AnthropicChatFormatter` / `AnthropicMultiAgentFormatter`
- `GeminiChatFormatter` / `GeminiMultiAgentFormatter`
- `DashScopeChatFormatter` / `DashScopeMultiAgentFormatter`
- `OllamaChatFormatter` / `OllamaMultiAgentFormatter`
- `DeepSeekChatFormatter` / `DeepSeekMultiAgentFormatter`
- `OpenAIResponseFormatter` / `OpenAIResponseMultiAgentFormatter`
- `Moonshot*`、`XAI*`

### Multi-agent formatter

多数 provider 不支持连续 assistant/assistant 消息。AgentScope 的 `*MultiAgentFormatter` 将多 agent 历史合并为带 `<history>` 标签的 user-side 内容，同时保留工具调用序列的结构化格式。这是多 agent 消息兼容的核心。

### 多模态 tool result 投影

`FormatterBase.convert_tool_result_to_string()` 会检查 formatter 支持的 media type：

- 支持的 data block：从 tool result 中提升为后续 user message 的多模态内容，并插入 identifier 提醒
- URL source 不支持时：把 URL 写入文本提醒
- Base64 source 不支持时：写入临时文件并把本地路径写入文本提醒

这让 provider 不支持 tool-result multimodal 时仍能把信息投影进模型可理解的格式。

---

## 关键代码路径

- `src/agentscope/agent/_agent.py` — `_prepare_model_input()` / `_get_system_prompt()`
- `src/agentscope/middleware/_base.py` — `on_system_prompt`
- `src/agentscope/formatter/_formatter_base.py` — formatter 基类、message grouping、tool result projection
- `src/agentscope/formatter/_openai_formatter.py` — OpenAI chat/multi-agent formatter
- `src/agentscope/formatter/_anthropic_formatter.py`
- `src/agentscope/formatter/_gemini_formatter.py`
- `src/agentscope/formatter/_dashscope_formatter.py`
- `src/agentscope/formatter/_openai_response_formatter.py`

---

## 设计亮点

### 1. Prompt 组装与 provider formatting 分层

system prompt、summary、context、tools 的选择在 agent 层完成；provider API 差异在 formatter 层处理。这个边界避免把 OpenAI/Anthropic/Gemini 的格式差异散落到主循环。

### 2. System prompt middleware 可按 session 动态注入

`on_system_prompt` 可用于插入租户策略、workspace 说明、实验性 guardrail 或观测标记。它是字符串 transformer，调试成本低于包裹整个模型调用。

### 3. MultiAgentFormatter 是分布式 agent 的必要适配层

当多个 agent 共享同一会话历史时，formatter 负责把“多个 agent 发言”投影为目标模型 API 能接受的消息形态。这个设计可以直接迁移到 agent-os 的 Web distributed agent。

### 4. DataBlock 支持与 model card 对齐

`FormatterBase.input_types` 和 `supported_input_media_types` 让 formatter 根据模型能力过滤图片/音频等 data block，而不是盲目发送 provider 不支持的内容。

---

## 局限性

### 1. Formatter 不再承担自动截断

当前 formatter 只做格式转换；token 预算和压缩由 agent/model 层处理。若某个 provider 需要 API-format 后的精确 token 计数，需要模型子类或上层 context manager 自行实现。

### 2. Multi-agent 历史压缩损失结构

把多 agent 发言合并进 `<history>` 文本会丢失原始消息边界和部分多模态顺序信息。角色强区分或需要精确审计的场景，应保留原始 event log 作为旁路。

### 3. Base64 fallback 会产生本地临时文件

不支持的多模态 base64 会被 formatter 写到本地临时文件。分布式部署时，本地路径对其他进程或远程沙箱不可用，应改成 workspace/object-store URI。

### 4. Capability 主要是 formatter 侧过滤

formatter 会 warning/skip 不支持的 media type，但并不会自动选择替代模型或重试。生产系统需要在 model routing 层提前校验 capability。

---

## 对 agent-os 的借鉴

agent-os 可以将 prompt 系统拆成三个 ABC：

- `SystemPromptBuilder`：组装基础指令、workspace、skills、memory、policy
- `PromptMiddleware`：按 session/tenant 顺序改写 system prompt
- `MessageFormatter`：把内部 `MessageBlock` 投影到 provider API

多 agent Web 形态应显式提供 `MultiAgentFormatter`，不要把 agent 名称简单塞进文本后交给每个业务方自行处理。

---

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
- 核心文件：
  - `src/agentscope/agent/_agent.py`
  - `src/agentscope/middleware/_base.py`
  - `src/agentscope/formatter/_formatter_base.py`
  - `src/agentscope/formatter/_openai_formatter.py`
  - `src/agentscope/formatter/_anthropic_formatter.py`
  - `src/agentscope/formatter/_gemini_formatter.py`
  - `src/agentscope/formatter/_dashscope_formatter.py`
  - `src/agentscope/formatter/_openai_response_formatter.py`
