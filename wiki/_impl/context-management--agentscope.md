---
title: "context-management——agentscope"
category: L2
parent: "[[context-management]]"
source: "agentscope"
source_version: "v2.0.1-11-g0e5418e8"
concept: "context-management"
created: "2026-04-15"
updated: "2026-06-10"
confidence: high
---

## 概述

AgentScope Python 2.x 的 context 管理收敛在 `AgentState` 和 `Agent.compress_context()`：会话内未压缩消息保存在 `state.context`，压缩摘要保存在 `state.summary`，模型输入由 `_prepare_model_input()` 组装 system prompt、summary、context、tools 和 skill instructions。触发阈值由 `ContextConfig.trigger_ratio * model.context_size` 控制，压缩由当前 chat model 的 `generate_structured_output()` 生成结构化摘要。

当前源码中未找到旧页引用的 `agentscope/token/`、`TruncatedFormatterBase`、`MemoryBase`、`_react_agent.py`、`rag/` 目录。旧版“token counter 层 + formatter 截断层 + memory 标记压缩层”的结论不再代表 2.x 主线。

---

## 架构分析

### 数据结构

`AgentState` 是 context 的核心容器：

| 字段 | 作用 |
|---|---|
| `session_id` | 当前 session id |
| `summary` | 被压缩历史的结构化摘要，送入模型输入 |
| `context` | 未压缩的 `Msg` 列表 |
| `reply_id` / `cur_iter` | 当前 reply 和 reasoning-acting 迭代状态 |
| `permission_context` | 工具权限上下文 |
| `tool_context` | 工具组激活状态和文件读取 cache |
| `tasks_context` | task/todo 状态 |

### 压缩触发

`Agent.compress_context()` 先走 `on_compress_context` middleware chain，再进入 `_compress_context_impl()`。核心流程：

1. `_prepare_model_input()` 组装当前模型输入
2. `model.count_tokens(messages, tools)` 估算输入 token
3. 若低于 `ContextConfig.trigger_ratio * model.context_size`，直接返回
4. 调用 `_split_context_for_compression()` 按 `reserve_ratio` 保留最近 context
5. 在 system prompt、已有 summary、待压缩 messages 后追加 `compression_prompt`
6. 用 `model.generate_structured_output(..., structured_model=summary_schema)` 生成摘要
7. 将摘要写入 `state.summary`
8. 若存在 `offloader`，把被压缩 messages 落到 workspace，并在 summary 中加入引用提醒
9. 清理不再保留的 read-file cache，最后用保留段替换 `state.context`

### 工具结果控制

AgentScope 2.x 对大工具结果不是靠 formatter 截断，而是在工具执行后调用 `_split_tool_result_for_compression()`。如果 tool result 超过 `ContextConfig.tool_result_limit`：

- 保留前段可放入 context 的结果
- 对超出部分生成提示，要求 agent 必要时通过 offload 路径读取
- 若 workspace/offloader 可用，工具结果可落到 workspace-accessible storage

这让“普通对话 context 压缩”和“单次工具结果过大”成为两个路径。

### Offloader / Workspace

`WorkspaceBase` 同时承担 workspace 和 offloader 角色。Local workspace 可把 compressed context 或 tool result 写入本地文件，Docker/E2B 则可把这套路径迁移到容器或远程沙箱。

---

## 关键代码路径

- `src/agentscope/state/_state.py` — `AgentState.summary` / `context` / `tool_context`
- `src/agentscope/agent/_config.py` — `ContextConfig`、`SummarySchema`、`tool_result_limit`
- `src/agentscope/agent/_agent.py` — `compress_context()` / `_compress_context_impl()`
- `src/agentscope/agent/_agent.py` — `_split_context_for_compression()`
- `src/agentscope/agent/_agent.py` — `_split_tool_result_for_compression()`
- `src/agentscope/model/_base.py` — 默认 `count_tokens()` 和 `generate_structured_output()`
- `src/agentscope/workspace/_offload_protocol.py` — `Offloader` 协议
- `src/agentscope/workspace/_local_workspace.py` — local context/tool-result offload

---

## 设计亮点

### 1. Context 状态集中在 `AgentState`

摘要、未压缩消息、当前 reply、权限和工具 cache 都在同一个 Pydantic state 中，适合 Web session 在每轮结束后整体持久化。对比旧式散落在 memory/formatter/token counter 的路径，当前 2.x 更像服务端 session runtime。

### 2. 压缩摘要是结构化输出

`SummarySchema` 将摘要拆成 task overview、current state、important discoveries、next steps、context to preserve 五段，并给每段设置长度约束。摘要不是自由文本，恢复工作时更容易保持任务导向。

### 3. 压缩与 offload 组合

被压缩的原始 messages 可以通过 `offloader.offload_context()` 落到 workspace，再把路径写进 summary。这个模式兼顾 token 控制和证据回链，适合本地代理和 Web distributed workspace 共享。

### 4. Read file cache 跟随 context 清理

`_clear_unreserved_read_cache()` 会根据保留 context 中仍引用的文件路径清理 `ToolContext.read_file_cache`。这避免 context 已压缩但旧文件 cache 仍无限增长。

### 5. 大工具结果单独限流

`tool_result_limit` 处理单次 payload 爆炸，不必等整轮 context 达到压缩阈值。对文件读取、搜索、命令输出密集的 agent 更实用。

---

## 局限性

### 1. 默认 token counting 是粗估

`ChatModelBase.count_tokens()` 默认把文本 UTF-8 字节数除以 4，并把 tools schema JSON 拼入估算。子类可以 override，但框架主路径不是多 tokenizer 精确计数。

### 2. 压缩在 reasoning 前触发，工具执行中途不会再次全局压缩

`_reply_impl()` 在进入 reasoning 前调用 `compress_context()`。若当轮产生大量工具结果，主要依赖 tool-result limit 和 offload，完整 context 压缩要等下一轮 reasoning 前。

### 3. 摘要覆盖式更新

`state.summary` 每次压缩被新摘要覆盖，虽然压缩 prompt 会带上旧 summary，但框架不保存摘要版本历史。若摘要质量下降，需要依赖 offloaded 原文或外部日志排查。

### 4. Offload 可用性依赖 workspace

没有 offloader 时，压缩只保留 summary 和最近 context；原文不会自动进入持久 evidence store。Web 形态需要明确 workspace/object-store 策略。

### 5. 工具结果压缩仍需模型自觉召回

大结果被截短/offload 后，agent 是否继续读取细节取决于提示和可用工具。生产系统应在工具层增加 per-turn recall budget 和 evidence 引用校验。

---

## 对 agent-os 的借鉴

agent-os 可以直接借鉴这条抽象边界：

- `ContextManager`：负责阈值、摘要 schema、reserve 策略
- `ContextOffloader`：负责把被压缩原文落到 local file / object store / DB
- `ToolResultProjector`：负责单次大工具结果的截短、offload 和可召回 handle
- `AgentState`：只保存 summary、active context、tool cache、permission state

本地代理实现可用文件 offload；Web/distributed 实现应把 offload 换成 object store + DB index，并把 evidence URI 纳入权限模型。

---

## 来源

- 项目：`/Users/neo/Desktop/project/git/agentscope`
- 版本：`v2.0.1-11-g0e5418e8`
- 核心文件：
  - `src/agentscope/state/_state.py`
  - `src/agentscope/agent/_agent.py`
  - `src/agentscope/agent/_config.py`
  - `src/agentscope/model/_base.py`
  - `src/agentscope/workspace/_offload_protocol.py`
  - `src/agentscope/workspace/_local_workspace.py`
