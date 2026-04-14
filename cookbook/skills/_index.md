---
section: skills
description: Skill 设计的实操参考，覆盖 5 种类型的结构模板、案例分析和反模式
---

# Skills Cookbook

> Skill 是 LLM 的技能包：轻量、按需加载、指导行为而非替代判断。

## 先搜索，再创建

**在创建任何新 skill 之前，必须先搜索现有 skill：**

```bash
npx skills find <query>
```

或访问 https://skills.sh/ 浏览。创建重复 skill 是最常见的浪费。只有在确认以下条件后才自己写：
- 搜索结果无匹配
- 现有 skill 太通用/语言不对/场景不符
- 差异足够大，无法通过简单修改现有 skill 解决

## 板块导航

| 板块 | 内容 | 链接 |
|------|------|------|
| **Patterns** | 5 种 skill 类型的结构模式和设计原则 | [patterns/](_index.md) |
| **Templates** | 可直接复制的 skill 文件模板（带 placeholder） | [templates/](_index.md) |
| **Anti-patterns** | 常见 skill 设计错误及修复方法 | [anti-patterns/](_index.md) |

## 5 种 Skill 类型速览

| 类型 | 特征 | 适用场景 | Pattern 页面 |
|------|------|---------|------------|
| **纯文本型** | 只有 SKILL.md，纯知识/规则 | 编码规范、设计原则、流程指南 | [pure-text-skill](patterns/pure-text-skill.md) |
| **多文档导航型** | SKILL.md 是入口，子文档按需 Read | 内容量大、多执行路径 | [multi-doc-navigation](patterns/multi-doc-navigation.md) |
| **Agent Dispatch 型** | skill 描述如何构造和派发 agent | 需要 subagent 执行复杂任务 | [agent-dispatch](patterns/agent-dispatch.md) |
| **外部 CLI 型** | 教 LLM 使用特定 CLI 工具 | 封装 gh、vercel、npx 等工具 | [cli-integration](patterns/cli-integration.md) |
| **脚本/Hook 型** | 带 shell 脚本，事件触发自动注入 | 持续提醒、自动化检测 | [hook-triggered](patterns/hook-triggered.md) |

## 与 wiki 的关系

cookbook/skills/ 是"怎么写 skill"的实操层，wiki/ 中的 Extension Model 概念页是"为什么这样设计"的原理层。
