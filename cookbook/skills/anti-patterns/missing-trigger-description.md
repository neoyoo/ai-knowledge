---
anti-pattern: missing-trigger-description
category: skill-design
tags: [skill, trigger, description, discoverability]
severity: high
related_patterns: [skill-metadata, use-when-convention, trigger-design]
---

# 触发描述缺失

> description 字段描述的是 skill 的内容，而非触发场景，导致 LLM 永远不知道该在什么时候使用它。

## 症状

- 安装了 skill，但 LLM 从不主动使用它
- 用户明确说出 skill 相关话题时，LLM 也不触发
- 用户必须手动说 "use the X skill" 才能激活
- skill 在 `skills.sh` 上展示良好，但在实际对话中形同虚设

## 错误示例

**错误写法——描述 skill 的内容：**

```yaml
---
skill: react-best-practices
description: "Contains best practices for React development including hooks, components, and performance."
---
```

```yaml
---
skill: git-commit-conventions
description: "A guide to writing good git commit messages following conventional commits spec."
---
```

这两个 description 回答的是"这个 skill 里有什么"，而不是"什么时候该用这个 skill"。

**正确写法——描述触发场景：**

```yaml
---
skill: react-best-practices
description: "Use when building React components, reviewing React code, asked about React patterns, hooks usage, or React performance optimization."
---
```

```yaml
---
skill: git-commit-conventions
description: "Use when writing a commit message, asked to commit changes, reviewing commit history, or explaining commit message conventions."
---
```

## 为什么有害

LLM 通过 description 做两件事：

1. **相关性判断**：当前对话是否需要这个 skill？
2. **激活时机决策**：现在是不是应该加载它？

如果 description 描述的是 skill 的内容（"Contains..."、"A guide to..."），LLM 得到的信息是"这个文件里有什么"，而不是"这个文件在什么情况下有用"。LLM 无法从内容描述推断触发时机——它不会去猜"含有 React 最佳实践的 skill 应该在用户问 React 时用"。

这不是 LLM 能力问题，是信息缺失问题。description 字段的设计目的就是提供触发信号，写错了等于把这个信号丢掉了。

## 正确做法

**规则：description 必须以 "Use when" 开头，后接 2-5 个具体触发场景。**

场景描述要具体到行为，不要宽泛到领域：

| 太宽泛（错） | 足够具体（对） |
|------------|-------------|
| "deployment related" | "user asks to deploy to production or staging" |
| "testing topics" | "writing unit tests, setting up test infrastructure, or debugging test failures" |
| "React development" | "building React components, reviewing React code, asked about hooks or React performance" |

**完整模板：**

```yaml
---
skill: <name>
description: "Use when <scenario-1>, <scenario-2>, or <scenario-3>."
---
```

**多场景拼接示例：**

```yaml
description: "Use when writing database queries, optimizing slow queries, designing schema, or asked about SQL best practices or ORM usage."
```

**注意事项：**
- 场景之间用逗号或 "or" 连接，不要换行
- 每个场景尽量包含动词（"writing"、"reviewing"、"asked about"、"debugging"）
- 避免品牌名作为唯一触发词（"Use when React" 太短，LLM 可能忽略）

## 修复检查清单

- [ ] description 是否以 "Use when" 开头？
- [ ] 是否列出了 2-5 个具体触发场景？
- [ ] 每个场景是否包含具体动词（不是只有名词/领域词）？
- [ ] 是否覆盖了用户可能的不同表述方式（如："writing X"、"reviewing X"、"asked about X"）？
- [ ] 安装后是否测试过：在对话中自然提及相关场景，skill 是否自动触发？
