---
title: "Prompt System — OpenHarness"
category: L2
parent: "[[prompt-system]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述
OpenHarness 的 prompt 系统通过 `build_runtime_system_prompt()` 在每次 `submit_message()` 前同步重建完整 system prompt，按固定顺序拼接七个独立 section：基础环境信息、可选的 fast-mode/effort/passes 推理设置、skills 索引、CLAUDE.md 内容（从 cwd 向上遍历收集）、issue/PR 上下文文件、MEMORY.md 入口、以及通过启发式 token 匹配检索到的相关记忆文件。

## 架构分析
### Section 组装顺序
七个 section 按固定顺序依次追加，每段之间用 `\n\n` 连接：
1. 基础 system prompt，包含环境信息（OS、shell、cwd、日期等）
2. Fast-mode 开关，以及 `effort` / `passes` 参数注入为 `# Reasoning Settings` section
3. 可用 skills 索引（从 skill 注册表动态生成）
4. CLAUDE.md 内容：从当前工作目录向上逐级查找，同时收集 `.claude/rules/*.md`
5. Issue 上下文文件与 PR 评论文件（若存在）
6. MEMORY.md 入口文件
7. 启发式 token 匹配检索出的相关记忆文件内容

### 关键代码路径
- `openharness/prompts/context.py` — `build_runtime_system_prompt()` 主入口，协调各 section 组装
- `openharness/prompts/system_prompt.py` — 基础 system prompt 与环境信息模板
- `openharness/prompts/claudemd.py` — CLAUDE.md 文件遍历与读取逻辑
- `openharness/prompts/environment.py` — 运行时环境元数据（OS、shell、日期等）收集

## 设计亮点
- `effort` + `passes` 作为 `# Reasoning Settings` section 注入 prompt，是 extended thinking / budget tokens 的轻量平替，无需模型侧特殊支持即可控制推理深度
- CLAUDE.md 向上目录遍历 + `.claude/rules/*.md` 支持项目级与全局级规则叠加，继承语义清晰
- 记忆检索采用启发式 token 匹配而非向量检索，实现简单、无外部依赖

## 局限性
- 无 prompt caching 或 cache-control headers，每轮完整重建会产生重复 token 费用
- 同步重建（无 async 优化），在记忆文件较多时可能成为延迟瓶颈
- 启发式 token 匹配召回质量依赖关键词覆盖，语义相关但用词不同的记忆会被漏掉

## 来源
- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
