---
title: Prompt System
aliases: [prompt 系统, prompt composition, dynamic prompting]
category: L1
created: 2026-04-06
updated: 2026-04-08
relations:
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[context-management]]"
    type: uses
sources: [claude-code, openharness, deer-flow, hermes-agent]
---

## 一句话定义

Agent 运行时中负责动态组装发给模型的 prompt 的子系统，不是静态模板。

## 核心问题

- 怎么把系统指令、用户输入、工具结果、历史对话组装成一个 prompt？
- 怎么根据运行时状态动态调整 prompt 内容？
- 怎么在 token 限制内取舍？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|-------------|
| 核心设计 | 分层认知控制框架，通过 `buildEffectiveSystemPrompt()` 按优先级动态装配六层 prompt，将 prompt 从静态模板升格为运行时策略层 | 通过 `build_runtime_system_prompt()` 在每次 `submit_message()` 前同步重建完整 system prompt，按固定顺序拼接七个独立 section（环境信息、推理设置、skills 索引、CLAUDE.md、issue/PR 上下文、MEMORY.md、记忆文件） | 分段组装（基础指令 + `<skill_system>` 技能索引 + `<memory>` 记忆注入 + 可选 SOUL.md 人格层），技能懒加载——提示里只放技能名/描述索引，任务匹配时再按需加载完整 SKILL.md，大幅压缩默认提示 token 占用 | **稳定缓存前缀 + 动态 ephemeral 追加**：`_build_system_prompt()` 在 session 开始时一次性构建 9 层装配流水线并缓存，每轮追加 ephemeral 层和 per-turn 记忆注入（注入到 messages 流而非 system prompt），系统 prompt 前缀全程不变，prefix cache 命中率接近 100% |
| 关键特点 | 静态/动态 section 分离 + `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 缓存感知设计；失败模式前置规则（伪完成/越权/过度设计压制）；coordinator 模式独立 prompt 实现完整认知切换 | `effort` + `passes` 作为 `# Reasoning Settings` section 注入 prompt，轻量替代 extended thinking；CLAUDE.md 向上目录遍历 + `.claude/rules/*.md` 支持全局与项目级规则叠加；记忆检索采用启发式 token 匹配，无外部依赖 | 技能懒加载：技能数量与提示 token 开销完全解耦；SOUL.md 人格层：人格与逻辑指令解耦，自定义 agent 可替换人格；tiktoken 精确预算：记忆注入用精确 token 计数而非字符估算 | per-model 执行协议：针对通用/OpenAI(GPT)/Google(Gemini) 三套独立执行纪律，靶向压制各家模型已知失败模式；SOUL.md persona-as-file：agent 身份做成可替换文件，runtime 不改代码即可托管完全不同人格；三层注入防护（内容正则 + 路径黑名单 + 注入量限制） |
| 局限 | 动态 section 持续膨胀，跨 section 语义冲突无工具检测；工具 prompt 与主 prompt 之间缺乏显式语义协调机制 | 无 prompt caching，每轮完整重建产生重复 token 费用；同步重建在记忆文件较多时可能成为延迟瓶颈 | 无提示缓存边界优化（每轮全量重新计算）；无推理提示注入（无 effort/passes 控制）；技能懒加载每次 read_file 消耗一个工具调用轮次 | Skin 视觉主题与 SOUL.md 人格两套系统各自独立，无同步机制，可能产生 banner 与 prompt 人格不一致；context files 取一不合并（`.hermes.md` > `AGENTS.md` > `CLAUDE.md` > `.cursorrules`），多规范共存时低优先级文件被静默忽略；注入防护为正则匹配，语义等价绕过无法识别 |

## 设计权衡

### 方案对比

| 方案 | 适用场景 | 优势 | 劣势 | 代表实现 |
|------|---------|------|------|---------|
| **A. 静态模板** | 单一用途 bot、原型验证 | 简单可预测，易于版本控制，prompt caching 命中率高 | 无法适应运行时上下文，功能复杂后难以维护 | 早期 chatbot、专用工具 |
| **B. 动态组装（Section-based）** | 多能力通用 agent、可配置行为 | 模块化，每个 section 独立维护，可按状态开关功能 | Section 间语义冲突不易发现，整体难以全量测试 | Claude Code `buildEffectiveSystemPrompt()`、OpenHarness `build_runtime_system_prompt()` |
| **C. 冻结前缀 + ephemeral 追加** | 长 session、成本敏感、多平台 gateway | Session 内 prefix cache 命中率接近 100%，推理成本大幅降低；ephemeral 层每轮独立，不污染缓存 | 不适合需要在同一 session 内频繁切换 system prompt 的场景；ephemeral 层不写入轨迹，调试时需额外注意 | Hermes Agent `_build_system_prompt()` + `ephemeral_system_prompt` |
| **D. 元提示（Meta-prompting）** | 研究实验、prompt 优化场景 | 最大适应性，可针对任务自动优化 prompt | 需两次 LLM 调用，prompt 非确定性，调试极难 | AutoPrompt、DSPy 优化管道 |

### 场景决策指南

- **单一用途 bot / 客服 / 专用 CLI** → 静态模板。需求稳定时不必引入动态组装的复杂度，优先保持 prompt caching 命中率。
- **通用开发助手 / 多角色 agent** → 动态组装（Section-based）。将 system prompt 拆为独立 section（环境信息、推理设置、规则、记忆），按运行时状态选择性装配；同时显式划分静态/动态边界，让不变部分命中 cache。
- **长 session / 多平台 gateway / API 成本敏感** → 冻结前缀 + ephemeral 追加（方案 C）。session 开始时一次性构建并缓存完整 system prompt；动态记忆注入到 messages 流而非 system prompt，保证前缀稳定。适合需要跨 WhatsApp / Telegram / cron 等多个平台分发同一 agent 的场景。
- **跨模型（Claude / GPT / Gemini）部署** → 维护 per-model 执行协议层。OpenAI 模型有"过早停止"失败模式，Google 模型有格式合规问题，通用纪律文本无法覆盖，需针对各家模型各自定制执行引导文本（Hermes Agent 的做法）。
- **可替换人格 / 品牌定制 bot** → persona-as-file（SOUL.md）。把 agent 身份做成文件而非代码常量，允许在不改代码的情况下彻底替换人格；注意 Skin（视觉）与 SOUL.md（语言人格）需要同步维护，否则 banner 与 prompt 人格会产生不一致体验。
- **研究 / prompt 优化流水线** → 元提示。仅在可以接受延迟和非确定性的离线场景使用；生产系统不推荐。
- **多模式切换（普通 vs coordinator）** → 方案 B 变体：不同角色用完全独立的 prompt builder，而非在一套 prompt 里用条件分支——Claude Code coordinator 模式即采用此策略。

### 常见陷阱

1. **Section 互相矛盾**：动态组装时，一个 section 说"简洁回答"，另一个说"详细解释"，模型行为不可预测。对策：section 入库前做语义冲突检查，或设置 section 优先级覆盖规则。
2. **Lost-in-the-middle**：prompt 过长时模型对中间内容注意力衰减。对策：重要规则放首尾，将"高优先级失败模式压制规则"置于 prompt 开头（Claude Code 的做法）。
3. **Cache 浪费**：没有静态/动态边界，每轮都全量重建 prompt，静态部分反复计算 token 费用。对策：参照 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 模式，把不变内容锁在 cache boundary 之前；或采用 Hermes Agent 的冻结前缀策略，将记忆/上下文注入到 messages 流而非 system prompt。
4. **全量替换 vs 追加混淆**：替代性角色 prompt 应全量替换默认 prompt，域增强型 prompt 应追加——混淆两者会导致指令叠加冲突或意外覆盖原有行为。
5. **通用执行纪律对特定模型失效**：对 GPT 说"请使用工具"不如用 XML 标签结构列出强制规则；对 Gemini 说"请用绝对路径"不如把 OS 操作规范独立成专属协议块。多模型部署时需维护 per-model 执行引导，而非依赖单一通用文本。
6. **注入防护单层过于脆弱**：仅靠正则关键词检测可被语义等价写法或 Unicode 零宽字符绕过。生产系统建议三层纵深：内容级（正则 + 不可见字符扫描）→ 路径级（敏感目录黑名单）→ 量级（外部注入占比上限），任意一层命中即阻断并明确提示，不要静默失败。

## L2 详情

- [[prompt-system--claude-code]]
- [[prompt-system--openharness]]
- [[prompt-system--deer-flow]]
- [[prompt-system--hermes-agent]]
