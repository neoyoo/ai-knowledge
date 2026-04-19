# Ingest Review: agentscope

**Ingest 时间**: 2026-04-15
**源路径**: /Users/neo/Desktop/project/git/agentscope
**源版本**: 0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12
**KB 路径**: /Users/neo/Desktop/project/git/ai- knowledge
**状态**: READY_TO_MERGE

## 概览

| 指标 | 数值 |
|------|------|
| L2 文件数 | 12 |
| L1 patch 数 | 12 |
| 覆盖率 | 90.5% (PASS) |
| 自动修复 | 2 |
| 待确认问题 | 0 |
| 阻断问题 | 0 |

## 产出文件清单

### L2 文件（12 个）

| 文件 | confidence |
|------|-----------|
| wiki/_impl/query-loop--agentscope.md | high |
| wiki/_impl/prompt-system--agentscope.md | high |
| wiki/_impl/memory-system--agentscope.md | high |
| wiki/_impl/context-management--agentscope.md | high |
| wiki/_impl/runtime-state--agentscope.md | high |
| wiki/_impl/session-recovery--agentscope.md | high |
| wiki/_impl/hooks--agentscope.md | high |
| wiki/_impl/mcp-skills--agentscope.md | high |
| wiki/_impl/tool-system--agentscope.md | high |
| wiki/_impl/multi-agent--agentscope.md | high |
| wiki/_impl/channel-remote--agentscope.md | high |
| wiki/_impl/evaluation-observability--agentscope.md | high |

### L1 Patch（12 个）

| 文件 |
|------|
| wiki/query-loop.md.patch |
| wiki/prompt-system.md.patch |
| wiki/memory-system.md.patch |
| wiki/context-management.md.patch |
| wiki/runtime-state.md.patch |
| wiki/session-recovery.md.patch |
| wiki/hooks.md.patch |
| wiki/mcp-skills.md.patch |
| wiki/tool-system.md.patch |
| wiki/multi-agent.md.patch |
| wiki/channel-remote.md.patch |
| wiki/evaluation-observability.md.patch |

### 分析文件

- drafts/agentscope/analysis/module-map.md
- drafts/agentscope/analysis/coverage-report.md

## 自动修复清单

**[AUTO-1]** `query-loop--agentscope.md` frontmatter title 分隔符不一致
- 位置：第 2 行 `title: "query-loop — agentscope"`
- 问题：使用了 `" — "` (空格+破折号+空格)，与其余 11 个文件的 `"——"` (双em破折号) 不一致
- 修复：已将 `" — "` 改为 `"——"`

**[AUTO-2]** `tool-system--agentscope.md` 关键代码路径章节标题层级错误
- 位置：第 94 行 `### 关键代码路径`
- 问题：使用了 H3 (`###`)，但四项必含章节的标准层级为 H2 (`##`)；其余 11 个文件均使用 H2
- 修复：已将 `###` 改为 `##`

## 待确认问题清单

无待确认问题

## 阻断问题清单

无阻断问题

## 特别发现

**tune/ + tuner/ 模块未被 ontology 覆盖**

AgentScope 包含一套完整的 SFT/DPO/GRPO finetuning pipeline（`tuner/` 模块），当前 12 个 L1 概念均无对应覆盖。`tune/` 目录是旧实现（导入即报 ImportError，已废弃），`tuner/` 是真实实现，包含：

- 多种训练策略（SFT / DPO / GRPO）
- 与 AgentScope 工具系统的集成（工具调用轨迹用于训练数据生成）
- 阿里云 PAI 平台训练任务提交

这是 AgentScope 的**独特生产级能力**，与其他框架有本质差异（大多数 agent 框架不内置 finetuning pipeline）。建议 Neo 评估是否将其升为新 L1 概念（候选名：`model-finetuning` 或 `training-pipeline`）。

## 合并指南

1. **移动 L2 文件**：将 `drafts/agentscope/wiki/_impl/` 下 12 个文件复制到 `wiki/_impl/` 目录

   ```bash
   cp "drafts/agentscope/wiki/_impl/"*.md "wiki/_impl/"
   ```

2. **应用 12 个 L1 patch**：对每个 patch 文件，找到对应的 `wiki/` L1 页面并应用补充内容

   涉及页面：
   - `wiki/query-loop.md`
   - `wiki/prompt-system.md`
   - `wiki/memory-system.md`
   - `wiki/context-management.md`
   - `wiki/runtime-state.md`
   - `wiki/session-recovery.md`
   - `wiki/hooks.md`
   - `wiki/mcp-skills.md`
   - `wiki/tool-system.md`
   - `wiki/multi-agent.md`
   - `wiki/channel-remote.md`
   - `wiki/evaluation-observability.md`

3. **更新 frontmatter**：合并后检查各 L1 页面的 `sources` 字段，确保 agentscope 已加入来源列表，`updated` 字段更新为 `2026-04-15`

4. **清理 drafts**：合并完成后可归档或删除 `drafts/agentscope/` 目录

5. **tune/tuner 评估**：参考「特别发现」，决定是否补充 L1 概念页（非阻断，可异步处理）
