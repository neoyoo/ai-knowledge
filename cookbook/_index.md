# Cookbook

> Agent 开发百科全书的实操层。概念原理在 wiki/，动手做看这里。

## 板块

| 板块 | 路径 | 内容 |
|------|------|------|
| **Prompt Patterns** | [prompts/patterns/](prompts/patterns/_index.md) | 提示词设计模式：CoT、ReAct、Structured Output 等 20 个 pattern |
| Prompt Templates | [prompts/templates/](prompts/templates/_index.md) | 场景化可复制模板（计划中） |
| Prompt Anti-patterns | [prompts/anti-patterns/](prompts/anti-patterns/_index.md) | 常见提示词反模式（计划中） |
| **Tool Definitions** | [tools/definitions/](tools/definitions/_index.md) | 按功能分类的工具定义：10 个类别，含跨项目对比和最佳实践 |
| **Tool Patterns** | [tools/patterns/](tools/patterns/_index.md) | 工具设计与编排模式：设计原则、错误处理、权限模型 |

## 怎么用

1. 从板块导航找到方向
2. 读 _index.md 确定具体页面
3. 读页面的 frontmatter + 一句话摘要快速判断相关性
4. 按需读取具体 section

## 页面约定

- 每个页面都有 YAML frontmatter（pattern/category/tags/related_wiki）
- `> 一句话定义` 紧跟标题，用于快速判断相关性
- 所有模板带 `{{placeholder}}`，可直接复制使用
- wikilink 链接到 wiki/ 概念页获取设计原理
