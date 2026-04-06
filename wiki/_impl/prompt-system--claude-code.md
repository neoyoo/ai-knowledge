---
title: "Prompt System — Claude Code"
category: L2
parent: "[[prompt-system]]"
source: claude-code
source_version: "2.1.88"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述

Claude Code 的提示词系统不是一段静态 system prompt，而是一套与运行时强耦合的分层认知控制框架。它通过 `buildEffectiveSystemPrompt()` 按优先级动态装配六层 prompt——从全局默认层、环境注入层、模式切换层，到 agent 定义层、工具层和子系统层——每一层都有独立职责，最终拼装出当前 session 的完整执行协议。这套设计的核心价值不在于哪句话写得好，而在于它把 prompt 从"文案"升格成了"运行时策略层"。

## 架构分析

### 装配优先级与组合策略

核心装配逻辑在 `src/utils/systemPrompt.ts` 的 `buildEffectiveSystemPrompt()` 函数中。优先级从高到低：

1. `overrideSystemPrompt` — 最高级别，直接替换全部默认逻辑，用于外部强制注入
2. `coordinator system prompt` — coordinator 模式专用，完整切换工作范式
3. `agent system prompt` — 主线程 agent 定义的专属 prompt
4. `custom system prompt` — 用户通过 CLI `--system-prompt` 传入
5. `default system prompt` — 标准 coding agent 默认规则集
6. `appendSystemPrompt` — 轻量追加层，不推翻原框架

关键设计细节：agent prompt 在**普通模式**下替代默认 prompt（全量替换），在 **proactive 模式**下追加到默认 prompt 后面（领域增强）。这一区分体现了"替代性角色 prompt"与"域增强型 prompt"的设计意图。

### 默认 Prompt 的 Section 化结构

`src/constants/prompts.ts` 将默认 system prompt 拆成多个独立 section，每个 section 有明确职责边界：

**全局静态部分（可跨用户缓存）：**
- `getSimpleIntroSection()` — 定义基础身份：软件工程 agent，不是泛用聊天助手
- `getSimpleSystemSection()` — 权限、输出通道、系统标签、injection 防御规则
- `getSimpleDoingTasksSection()` — 编码任务行为协议（最关键的一段）
- `getActionsSection()` — 风险分级意识，destructive action 的确认边界
- `getUsingYourToolsSection()` — 工具使用策略：专用工具优先、并行调用、SubAgent 时机
- `getSimpleToneAndStyleSection()` — 输出风格：简洁、不用 emoji、源码引用格式
- `getOutputEfficiencySection()` — 输出成本控制：先答后解释、不旁白内部步骤

**动态部分（依赖当前 session）：**
- `getSessionSpecificGuidanceSection()` — 按当前启用工具动态生成操作策略补丁
- `computeEnvInfo()` / `computeSimpleEnvInfo()` — 工作目录、平台、shell、模型版本、知识截止时间
- `getLanguageSection()` — 会话语言偏好
- `getMcpInstructionsSection()` — 已连接 MCP server 的使用说明
- `getScratchpadInstructions()` — 临时文件策略
- `getFunctionResultClearingSection()` — 工具结果清除机制的告知

### 失败模式压制设计

`getSimpleDoingTasksSection()` 的核心价值在于针对 coding agent 高频失败模式的显式约束：

**过度工程化压制：** 明确禁止添加未要求的 feature、顺手重构、为一次性逻辑提取抽象、为"未来可能需要"预先设计。直接对冲大模型"高级工程感"倾向。

**伪完成压制：** 要求如实汇报测试失败、未验证状态，禁止把失败包装成完成。针对 coding agent 最危险的错误——错误完成汇报。

**越权行为压制：** 权限拒绝是运行时信号，不是普通错误。被拒绝的 tool call 不能原样重试。

**无上下文行动压制：** 多处明确要求先读文件、理解实现，再给修改建议。强制建立局部语义模型后才能进入编辑阶段。

### 缓存边界设计

`src/constants/prompts.ts` 中定义了 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 常量，将 prompt 明确分成两段：

- **边界前**：跨用户、跨组织可缓存的稳定内容（Intro、System、Doing tasks 等七个 section）
- **边界后**：依赖当前 session 的动态内容（env、language、MCP 等十余个 section）

这是 prompt infrastructure 思维：不只考虑行为效果，还考虑缓存命中率和请求成本。

### Coordinator Prompt — 模式级认知切换

`src/coordinator/coordinatorMode.ts` 的 `getCoordinatorSystemPrompt()` 不是"补一句你可以调 worker"，而是完整切换了工作范式：

- 明确角色边界：协调者，不是执行者
- 定义 worker 的使命分工：research / implementation / verification 三阶段
- 禁止把"理解结果"外包给 worker，必须先自行综合
- 规定 task notification 的消息语义
- 要求能并行就并行

这说明 coordinator 模式不是默认 prompt 的功能增强，而是一套独立的多智能体协作协议。

### 工具 Prompt 层与子系统 Prompt 层

各工具拥有独立 `prompt.ts`，工具不是裸 API 而是"被语言解释的能力对象"，模型会收到何时用、不要如何误用的语义提示。子系统同样如此——记忆、压缩等各自有独立 prompt，不同子任务由不同 prompt 接管，避免主 prompt 膨胀失控。

### 关键代码路径

- `src/utils/systemPrompt.ts` — `buildEffectiveSystemPrompt()`，装配入口，优先级决策树
- `src/constants/prompts.ts` — 默认 prompt 所有 section 函数，`SYSTEM_PROMPT_DYNAMIC_BOUNDARY`
- `src/coordinator/coordinatorMode.ts` — `getCoordinatorSystemPrompt()`，模式级 prompt 切换
- `src/tools/AgentTool/prompt.ts` — AgentTool 的专属 prompt
- `src/tools/BashTool/prompt.ts` — BashTool 的专属 prompt
- `src/tools/FileReadTool/prompt.ts` — FileReadTool 的专属 prompt
- `src/tools/FileEditTool/prompt.ts` — FileEditTool 的专属 prompt
- `src/services/SessionMemory/prompts.ts` — 记忆子系统 prompt
- `src/services/compact/prompt.ts` — 压缩子系统 prompt
- `src/QueryEngine.ts` — prompt 与 session 状态的最终装配现场
- `src/screens/REPL.tsx` — 交互式 session 的 prompt 构建调用点

## 设计亮点

- **Prompt-as-runtime**：通过优先级装配机制，让 prompt 成为运行模式的语言接口，而不是固定模板。不同模式（coordinator、agent、proactive）对应不同认知框架，切换模式即切换整套执行协议。
- **失败模式前置规则**：`Doing tasks` section 的约束明显来自真实工程事故沉淀，而不是理想化假设。把最昂贵的错误（伪完成、越权、过度设计）写进 prompt 而不是靠代码拦截，充分利用了 LLM 的语义理解能力。
- **缓存感知设计**：`SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 体现了 prompt infrastructure 思维——在设计 prompt 的同时就考虑缓存边界，是生产级工程的典型特征。
- **闭环一致性**：Prompt 中描述的规则（权限拒绝不重试、上下文会压缩、工具结果可能被清除）在运行时都有对应实现，prompt 不是空话而是真实约束的语言映射。

## 局限性

- **Section 越来越多的维护成本**：随着功能增加，动态 section 的数量持续膨胀（已超过十余个），跨 section 的语义冲突检测缺乏工具支持，靠人工维护一致性难以扩展。
- **Coordinator prompt 独立维护，与默认 prompt 的规则可能漂移**：两套 prompt 各自演化，没有显式的共享规则基础层，未来可能出现行为不一致。
- **动态边界后的内容不可缓存，但部分内容（如 language）实际上跨 session 稳定**：当前以"是否依赖 session"为粒度划分边界，粒度较粗，存在过度放弃缓存的情况。
- **工具 prompt 与主 prompt 之间没有显式的语义协调机制**：各工具 prompt 独立编写，如果工具使用策略在主 prompt 和工具 prompt 中描述不一致，模型行为可能受不可预测的影响。

## 来源

- 源码版本：Claude Code 2.1.88 (npm @anthropic-ai/claude-code)
- 分析深度：源码级
