---
title: "Prompt System — DeerFlow"
category: L2
parent: "[[prompt-system]]"
source: deer-flow
source_version: "2.0"
confidence: high
created: 2026-04-07
updated: 2026-04-07
---

## 概述

DeerFlow 使用分段组装的系统提示（section-based prompt assembly）：基础指令 + `<skill_system>` 技能索引 + `<memory>` 记忆注入 + 可选的 SOUL.md 人格层。最关键的设计是技能**懒加载**——提示里只放技能的 name/description 索引，当 agent 判断任务匹配某技能时，再通过 `read_file` 工具按需加载完整 SKILL.md，大幅压缩默认提示 token 占用。

## 架构分析

### 分段组装结构

系统提示由四个部分按顺序拼接：

1. **基础 agent 指令** — 角色定义、行为规范、输出格式
2. **`<skill_system>` 块** — XML 包裹的技能索引列表，每条仅含技能名 + 一句描述
3. **`<memory>` 块** — XML 包裹的记忆条目，按置信度降序排列
4. **SOUL.md 注入**（可选）— 自定义 agent 的人格/风格覆盖层，由 `config.yaml` 启用

### 懒加载技能系统

技能索引只提供名称和描述，完整实现不进提示。Agent 被指示在任务匹配时调用 `read_file` 加载对应的 `SKILL.md`。这将技能变成了运行时按需查阅的文档，而非静态注入的代码块。

`deerflow/skills/loader.py` 负责扫描技能目录、生成索引列表，输出结构类似：

```
- web_search: 搜索互联网获取实时信息
- code_execute: 在沙箱中运行 Python 代码
- ...
```

### Token 预算限制的记忆注入

记忆内容来自 `memory.json`，注入时受 token 预算约束（默认 2000 tokens），超出部分按置信度截断。Token 计数使用 `tiktoken` 精确统计，非字符估算。自定义 agent 可通过 `config.yaml` 调整预算上限。

### 自定义 Agent 限制层

`config.yaml` 可声明该 agent 可用的工具子集和技能子集，`make_lead_agent` 在构建时过滤掉不在列表内的工具，实现多租户场景下的能力隔离。

### 关键代码路径

- `deerflow/agents/lead_agent/agent.py` — `make_lead_agent()` 负责拼接所有提示段并初始化 agent
- `deerflow/skills/loader.py` — 扫描技能目录、生成 `<skill_system>` 索引内容
- `deerflow/agents/memory/updater.py` — 读取 memory.json、按置信度排序、执行 token 预算截断后输出 `<memory>` 块

## 设计亮点

- **懒加载技能**：技能索引 vs. 完整内容分离，默认提示大小与技能数量解耦，100 个技能和 10 个技能的提示开销几乎相同
- **SOUL.md 人格层**：人格与逻辑指令解耦，自定义 agent 可替换人格而不触动核心行为规范
- **tiktoken 精确预算**：记忆注入用精确 token 计数而非字符估算，在有限 context window 下利用率更高
- **XML 标签分区**：`<skill_system>` 和 `<memory>` 用 XML 标签明确划定语义边界，便于 LLM 区分不同信息来源

## 局限性

- **无提示缓存边界优化**：提示组装不考虑 prompt cache boundary（如 Anthropic 的 cache_control），每轮全量重新计算，缓存命中率低于 Claude Code 的设计
- **无推理提示（effort hints）**：不像 OpenHarness 那样在提示中注入 passes/effort 指令来控制推理深度
- **技能懒加载的隐性开销**：每次 `read_file` 调用消耗一个工具调用轮次，高频使用多个技能时累积延迟不可忽视
- **SOUL.md 缺乏校验**：人格注入内容无格式约束，自定义 agent 的行为一致性依赖 SOUL.md 编写质量

## 来源

- 源码版本：DeerFlow 2.0 (bytedance/deer-flow)
- 分析深度：源码级
