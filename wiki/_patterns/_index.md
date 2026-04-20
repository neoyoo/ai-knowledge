---
title: Architectural Patterns
category: index
updated: 2026-04-19
---

# 跨概念组合模式

> 不归任一个 L1 概念管辖的**可迁移架构组合**。出现在多个 L1 概念的交叉点，解决特定协作问题。

## 和其他层的关系

| 层 | 关注 | 声音 |
|---|---|---|
| `wiki/*.md` L1 | 单概念的各家实现对比 | 第三人称，客观 |
| `wiki/_impl/*.md` L2 | 某源某概念的具体实现 | 第三人称，深度 |
| `wiki/_insights/*.md` | **单源**的精彩设计抽出 | 第三人称，聚焦 |
| **`wiki/_patterns/*.md`** | **跨多个概念**的组合协议，可迁移到别项目 | 去身份化，架构语汇 |
| `practice/*.md`（拟） | 我们做过的实践沉淀 | 第一人称 |
| `ideas/*.md` | 好想法 inbox，等评估落地 | 任意声音 |

## 收录标准

一个候选必须满足：

1. **跨概念** — 横跨 ≥2 个 L1 概念（否则归入该 L1 或 `_insights/`）
2. **可迁移** — 协议/接口可以脱离某个具体源实现（不然就是单源 insight）
3. **有至少一个参考实现** — 纯推演的不收（放 `ideas/`）
4. **解决具体问题** — 不收"通用的设计原则"（那应归入 cookbook 或设计哲学）

## 现有 pattern

- [[tool-metadata-driven-context-lifecycle]] — LLM 主动驱动的 tool_result + 历史消息生命周期（四层互补防线）

## 待评估想法（ideas/ 中）

见 [../ideas/_index.md](../../ideas/_index.md)
