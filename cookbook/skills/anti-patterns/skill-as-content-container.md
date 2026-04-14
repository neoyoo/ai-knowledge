---
anti-pattern: skill-as-content-container
category: skill-design
tags: [skill, context-window, token-efficiency, progressive-disclosure]
severity: high
related_patterns: [quick-reference-table, progressive-disclosure, modular-skill-structure]
---

# Skill 当内容容器用

> 把所有知识都塞进一个 SKILL.md，导致 LLM 每次加载时吞噬大量 context，回答变慢、无关内容稀释注意力。

## 症状

- SKILL.md 超过 500 行
- LLM 回答该 skill 相关问题时明显变慢
- 一个 skill 同时覆盖多个独立技术栈（如 React + Vue + Angular）
- skill 加载后 context 使用量立即飙升
- LLM 偶尔把不相关章节的内容混入回答

## 错误示例

一个 `frontend-standards/SKILL.md`，塞入三套框架的完整规范，共约 800 行：

```markdown
# Frontend Standards

## React Patterns (250 lines)
### Component Structure
...
### Hooks Best Practices
...
### Performance Optimization
...

## Vue Patterns (250 lines)
### Options API vs Composition API
...
### Reactivity System
...

## Angular Patterns (300 lines)
### Module Architecture
...
### Dependency Injection
...
### Change Detection
...
```

用户问 "How should I structure a React component?" 时，LLM 仍然加载了 Vue 和 Angular 的全部内容。

## 为什么有害

1. **Context window 是稀缺资源**：LLM 的 context window 有限。一个 800 行的 skill 文件可能占掉 6000-10000 tokens，挤压其他重要上下文（用户代码、对话历史）的空间。

2. **注意力稀释**：LLM 注意力机制会被无关内容干扰。当用户问 React 问题时，Angular 和 Vue 的内容不仅没有帮助，还会增加 LLM 错误引用跨框架内容的概率。

3. **每次全量加载**：skill 被触发时，SKILL.md 整体加载进 context。没有按需加载机制——LLM 无法"只读前100行"。

4. **维护成本放大**：文件越大，不同章节之间越容易出现矛盾和重复，lint 和更新成本随文件大小非线性增长。

## 正确做法

SKILL.md 只做入口（< 100 行），包含 Quick Reference 表，告诉 LLM 去哪读子文档：

```
frontend-standards/
├── SKILL.md              ← 入口，< 100 行，Quick Reference 表
├── react-patterns.md     ← 按需加载：用户问 React 时才 Read
├── vue-patterns.md       ← 按需加载：用户问 Vue 时才 Read
└── angular-patterns.md   ← 按需加载：用户问 Angular 时才 Read
```

`SKILL.md` 示例：

```markdown
---
skill: frontend-standards
description: "Use when writing or reviewing frontend code with React, Vue, or Angular"
---

# Frontend Standards

## Quick Reference

| Framework | 何时使用 | 文档 |
|-----------|---------|------|
| React | 新项目默认选择，函数组件 + hooks | Read `react-patterns.md` |
| Vue | 已有 Vue 项目，或团队偏好选项式 API | Read `vue-patterns.md` |
| Angular | 企业级大型项目，强类型依赖注入 | Read `angular-patterns.md` |

## 使用方式

根据用户使用的框架，Read 对应子文档后再回答。
无法从上下文判断框架时，询问用户。
```

子文档各自只包含该框架的内容，按需读取，互不干扰。

## 修复检查清单

- [ ] SKILL.md 是否 < 200 行？（理想目标 < 100 行）
- [ ] SKILL.md 是否只包含 Quick Reference 表和导航说明，不包含具体规范内容？
- [ ] 是否按独立关注点拆分了子文档？（每个子文档只聚焦一个技术或场景）
- [ ] 子文档是否在 Quick Reference 表中被明确引用，告知 LLM 何时 Read？
- [ ] 每个子文档是否也控制在合理长度（< 300 行）？
