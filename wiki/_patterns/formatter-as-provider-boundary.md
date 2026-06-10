---
title: Formatter as Provider Boundary
aliases: [formatter boundary, provider adapter formatter, prompt formatter layer]
kind: pattern
created: 2026-05-02
updated: 2026-06-10
concepts_involved: [[prompt-system]], [[context-management]], [[multi-agent]], [[channel-remote]]
reference_implementations: [agentscope]
status: mature
---

## 一句话定义

把 provider API 差异、多 agent 消息兼容、tool_use/tool_result 配对和多模态 tool result 投影收敛到独立 formatter 层，让 agent loop 只处理统一的内部消息对象。

## 触发问题

同一个 agent 框架要接 OpenAI、Anthropic、Gemini、DashScope、A2A realtime 等接口时，消息格式差异会污染主循环：role 命名不同、tool result 位置不同、多模态块编码不同、assistant→assistant 连续消息是否允许也不同。

## 参与的概念

- [[prompt-system]] — formatter 是 prompt 组装的最后一公里
- [[context-management]] — formatter 输出会被 context/token 预算约束，但 AgentScope 2.x 的压缩触发已转到 agent/model 层
- [[multi-agent]] — multi-agent 历史需要折叠成 provider 可接受的消息形态
- [[channel-remote]] — A2A / realtime channel 可复用同一内部消息协议

## 核心设计

### 1. 内部消息对象保持 provider-neutral

Agent 内部只流转统一 `Msg` / content block。provider 细节不进入 query loop、memory、tool executor。

### 2. Formatter 是显式依赖

Agent 构造时注入 formatter。同一个 model 可以搭配不同 formatter：单 agent chat、多 agent chat、A2A bridge、带截断版本。

### 3. 保持 provider API 合法消息边界

Formatter 不只做字符串拼接，还负责把内部 `Msg`、tool use/result、data block 转成目标 provider 接受的合法消息序列。AgentScope 2.x 当前不再用 formatter 做自动截断；context 压缩由 `Agent.compress_context()` 和 `model.count_tokens()` 触发。

### 4. 多 agent 兼容在 formatter 层解决

当 provider 不允许连续 assistant 消息时，MultiAgentFormatter 把历史多 agent 发言折叠为带角色名前缀或 `<history>` 标签的 user 消息，保留最近工具链的结构化消息。

## 适用场景

- 多 provider agent 框架
- 同时支持单 agent 与 multi-agent
- 需要 provider-specific message formatting / multimodal projection
- 需要把同一 agent 暴露到 HTTP、WebSocket、A2A 等 channel

## 不适用场景

- 只接一个 provider、消息结构简单的内部 bot
- 输出必须保留完整多 agent 角色边界，不能接受历史折叠损失
- provider SDK 已经完全接管消息适配且不可替换

## 参考实现

AgentScope Python 2.x formatter 体系：

- `wiki/_impl/prompt-system--agentscope.md` — ChatFormatter / MultiAgentFormatter / A2AChatFormatter
- `wiki/_impl/context-management--agentscope.md` — context 压缩已转到 AgentState / Agent.compress_context / model.count_tokens
- `wiki/_impl/channel-remote--agentscope.md` — 当前主线是 FastAPI session REST/SSE，不再是旧 realtime / A2A bridge

## 迁移 checklist

- 定义 provider-neutral 的内部消息和 content block
- 每个 provider 单独实现 formatter，不在 query loop 写 `if provider == ...`
- context 压缩和 provider formatting 分层，避免 formatter 同时承担预算策略和 API 转换
- tool_use/tool_result 成对规则写成测试
- multi-agent 历史折叠要清楚标注角色名和时间顺序

## 常见陷阱

- formatter 只做格式转换，不负责 token 预算，导致 context-management 仍散落各处
- multi-agent 历史折叠丢失关键角色边界
- tool result 图片转成临时本地文件后，在分布式部署中不可访问
- capability 标记只写在 formatter 上，但框架没有自动降级

## 相关概念

- [[prompt-system]]
- [[context-management]]
- [[multi-agent]]
- [[channel-remote]]
