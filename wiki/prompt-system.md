---
title: Prompt System
aliases: [prompt 系统, prompt composition, dynamic prompting]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[context-management]]"
    type: uses
sources: [claude-code]
---

## 一句话定义

Agent 运行时中负责动态组装发给模型的 prompt 的子系统，不是静态模板。

## 核心问题

- 怎么把系统指令、用户输入、工具结果、历史对话组装成一个 prompt？
- 怎么根据运行时状态动态调整 prompt 内容？
- 怎么在 token 限制内取舍？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | 分层认知控制框架，通过 `buildEffectiveSystemPrompt()` 按优先级动态装配六层 prompt，将 prompt 从静态模板升格为运行时策略层 |
| 关键特点 | 静态/动态 section 分离 + `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 缓存感知设计；失败模式前置规则（伪完成/越权/过度设计压制）；coordinator 模式独立 prompt 实现完整认知切换 |
| 局限 | 动态 section 持续膨胀，跨 section 语义冲突无工具检测；工具 prompt 与主 prompt 之间缺乏显式语义协调机制 |

## 设计权衡

- **静态缓存 vs 动态适应**：Claude Code 选择了显式划分静态/动态边界（`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`），因为 prompt 基础设施必须在行为正确性和缓存命中率之间同时优化，纯动态会产生不必要的 API 成本。
- **全量替换 vs 追加增强**：agent prompt 在普通模式下全量替换默认 prompt，在 proactive 模式下追加——区分"替代性角色 prompt"与"域增强型 prompt"，允许不同场景用不同组合策略而不是强制统一。

## L2 详情

- [[prompt-system--claude-code]]
