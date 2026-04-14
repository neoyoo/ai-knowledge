---
section: skills/patterns
description: 5 种 skill 类型的结构模式和设计原则
---

# Skill Patterns

> 每种类型对应不同的 skill 设计需求。选错类型是 skill 设计最常见的起点错误。

## 类型导航

| Pattern | 一句话摘要 | 难度 | 典型案例 |
|---------|----------|------|---------|
| [纯文本型](pure-text-skill.md) | 只有 SKILL.md，纯知识/规则，最简单也最常用 | 初级 | brainstorming, dispatching-parallel-agents |
| [多文档导航型](multi-doc-navigation.md) | SKILL.md 是入口地图，子文档按需 Read | 中级 | writing-skills, requesting-code-review |
| [Agent Dispatch 型](agent-dispatch.md) | skill 描述如何构造和派发 agent，含 prompt 模板 | 高级 | subagent-driven-development |
| [外部 CLI 型](cli-integration.md) | 教 LLM 使用特定 CLI 工具，含决策树和命令参考 | 中级 | gh-cli, deploy-to-vercel |
| [脚本/Hook 型](hook-triggered.md) | 带 shell 脚本，事件触发自动向 LLM 注入提示 | 高级 | self-improvement |

## 选型指南

```
需要什么？
├── 告诉 LLM 知识/规则 → 纯文本型
│   └── 内容 > 200 行或多条路径 → 多文档导航型
├── 让 LLM 执行复杂任务（需要 subagent）→ Agent Dispatch 型
├── 教 LLM 使用外部工具/CLI → 外部 CLI 型
└── 需要自动触发（不等 LLM 主动调用）→ 脚本/Hook 型
```
