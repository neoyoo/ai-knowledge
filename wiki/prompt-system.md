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
sources: [claude-code, openharness]
---

## 一句话定义

Agent 运行时中负责动态组装发给模型的 prompt 的子系统，不是静态模板。

## 核心问题

- 怎么把系统指令、用户输入、工具结果、历史对话组装成一个 prompt？
- 怎么根据运行时状态动态调整 prompt 内容？
- 怎么在 token 限制内取舍？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | 分层认知控制框架，通过 `buildEffectiveSystemPrompt()` 按优先级动态装配六层 prompt，将 prompt 从静态模板升格为运行时策略层 | 通过 `build_runtime_system_prompt()` 在每次 `submit_message()` 前同步重建完整 system prompt，按固定顺序拼接七个独立 section（环境信息、推理设置、skills 索引、CLAUDE.md、issue/PR 上下文、MEMORY.md、记忆文件） |
| 关键特点 | 静态/动态 section 分离 + `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 缓存感知设计；失败模式前置规则（伪完成/越权/过度设计压制）；coordinator 模式独立 prompt 实现完整认知切换 | `effort` + `passes` 作为 `# Reasoning Settings` section 注入 prompt，轻量替代 extended thinking；CLAUDE.md 向上目录遍历 + `.claude/rules/*.md` 支持全局与项目级规则叠加；记忆检索采用启发式 token 匹配，无外部依赖 |
| 局限 | 动态 section 持续膨胀，跨 section 语义冲突无工具检测；工具 prompt 与主 prompt 之间缺乏显式语义协调机制 | 无 prompt caching，每轮完整重建产生重复 token 费用；同步重建在记忆文件较多时可能成为延迟瓶颈 |

## 设计权衡

### 方案对比

| 方案 | 适用场景 | 优势 | 劣势 | 代表实现 |
|------|---------|------|------|---------|
| **A. 静态模板** | 单一用途 bot、原型验证 | 简单可预测，易于版本控制，prompt caching 命中率高 | 无法适应运行时上下文，功能复杂后难以维护 | 早期 chatbot、专用工具 |
| **B. 动态组装（Section-based）** | 多能力通用 agent、可配置行为 | 模块化，每个 section 独立维护，可按状态开关功能 | Section 间语义冲突不易发现，整体难以全量测试 | Claude Code `buildEffectiveSystemPrompt()`、OpenHarness `build_runtime_system_prompt()` |
| **C. 元提示（Meta-prompting）** | 研究实验、prompt 优化场景 | 最大适应性，可针对任务自动优化 prompt | 需两次 LLM 调用，prompt 非确定性，调试极难 | AutoPrompt、DSPy 优化管道 |

### 场景决策指南

- **单一用途 bot / 客服 / 专用 CLI** → 静态模板。需求稳定时不必引入动态组装的复杂度，优先保持 prompt caching 命中率。
- **通用开发助手 / 多角色 agent** → 动态组装（Section-based）。将 system prompt 拆为独立 section（环境信息、推理设置、规则、记忆），按运行时状态选择性装配；同时显式划分静态/动态边界，让不变部分命中 cache。
- **研究 / prompt 优化流水线** → 元提示。仅在可以接受延迟和非确定性的离线场景使用；生产系统不推荐。
- **多模式切换（普通 vs coordinator）** → 方案 B 变体：不同角色用完全独立的 prompt builder，而非在一套 prompt 里用条件分支——Claude Code coordinator 模式即采用此策略。

### 常见陷阱

1. **Section 互相矛盾**：动态组装时，一个 section 说"简洁回答"，另一个说"详细解释"，模型行为不可预测。对策：section 入库前做语义冲突检查，或设置 section 优先级覆盖规则。
2. **Lost-in-the-middle**：prompt 过长时模型对中间内容注意力衰减。对策：重要规则放首尾，将"高优先级失败模式压制规则"置于 prompt 开头（Claude Code 的做法）。
3. **Cache 浪费**：没有静态/动态边界，每轮都全量重建 prompt，静态部分反复计算 token 费用。对策：参照 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 模式，把不变内容锁在 cache boundary 之前。
4. **全量替换 vs 追加混淆**：替代性角色 prompt 应全量替换默认 prompt，域增强型 prompt 应追加——混淆两者会导致指令叠加冲突或意外覆盖原有行为。

## L2 详情

- [[prompt-system--claude-code]]
- [[prompt-system--openharness]]
