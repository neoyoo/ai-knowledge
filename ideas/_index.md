---
title: Ideas Inbox
category: index
updated: 2026-04-19
---

# 想法 Inbox

> 好想法的起点。**按功能维度分文件**，每个文件内容纳多条相关想法，一条一节。定期 review 决定升级 / 孵化 / 死亡。

## 目的

- 把"脑子里闪过但还没落地"的东西落纸，防止遗失
- 给定期维护（kb-maintain）提供**可评估的 TODO 列表**
- 失败的想法也保留——失败理由本身是知识

## 组织方式

按**功能域**（domain）分文件，每个域文件内按 `## YYYY-MM-DD — 想法名` 组织多条想法。单条想法 20-30 行轻格式（核心想法 / 为啥值得 / 落地 / 风险 / 相关）。

## 现有功能域

| 文件 | 覆盖 | 活跃想法 |
|------|------|---------|
| [[kb-governance]] | 知识库治理、自进化、归档提升 | 3 条 |
| [[neoagent-designs]] | neoagent 待抽取的原创设计 | 2 条 |
| [[sandbox-security]] | 沙箱选型、下载安全、代理层架构 | 3 条 |
| [[web-service-architecture]] | Web 服务化 agent 的生命周期、prompt cache、持久化、存储分层 | 4 条 |
| [[wiki-structure]] | wiki 本身的结构调整 | 1 条 |

## 状态管理

每条想法在 section 开头标 `**status**:`：

```
inbox       →  incubating  →  promoted  →  (移出，落地到目标目录)
                 ↓
                dead  (保留原条目 + 死亡理由，不删)
```

| 状态 | 含义 |
|------|------|
| `inbox` | 刚落下，未评估 |
| `incubating` | 评估过，等时机/依赖成熟 |
| `promoted` | 已升级。应把该条从所属域文件删除（或移到 `## 已升级` 附录节记录链接） |
| `dead` | 不做。保留原条目 + 死亡理由 |

## 升级目标路径

| 想法性质 | 升级去向 |
|---------|---------|
| 跨概念架构模式 | `wiki/_patterns/` |
| 单源精彩设计 | `wiki/_insights/` |
| 我们做过的组合实践 | `practice/composite-patterns/`（待建） |
| 通用可复制模板 | `cookbook/*/templates/` |
| wiki 主干修改（补档、新章节）| 直接改对应 `wiki/*.md` |
| 项目决策 | `projects/*/decisions/`（已建） |

## 命名新域

当新想法不适合现有四个域：

1. 评估是否真的需要新域，还是可以归入某个已有域（域越少越好，管理成本低）
2. 如果确实是新域，加一个 `ideas/<domain>.md`，在本 index 加一行
3. 域名用 kebab-case，短名（2 词内）

## 已升级（归档）

- `tool-metadata-driven-context-lifecycle` — 2026-04-19 升级到 `wiki/_patterns/tool-metadata-driven-context-lifecycle.md`。原始灵感直接从 trip-os 对话产出，无独立 idea 页
- `agent-registry-discovery` — 2026-04-19 直接起草到 `wiki/agent-registry-discovery.md`（L1 概念页）。由 trip-os "分布式 subagent" 决策摩擦 + Neo Java 后端类比 + "Agent OS 基础支撑"视角共同催生。信息密度够未经过 inbox 阶段

## 已死亡（归档）

（空）
