---
section: skills/templates
description: 可直接复制的 skill 文件模板，带 {{placeholder}}
---

# Skill Templates

> 找到对应类型，复制模板，替换 {{placeholder}}，完成。

## 模板列表

| 模板 | 适用场景 | 对应 Pattern |
|------|---------|------------|
| [知识/规则型](knowledge-skill.md) | 编码规范、设计原则、概念参考、最佳实践 | 纯文本型 |
| [工作流型](workflow-skill.md) | code review、debugging、deployment 等有步骤的流程 | 纯文本型 / 多文档导航型 |
| [工具集成型](tool-integration-skill.md) | 封装 CLI 工具或外部服务（gh、vercel、npx 等） | 外部 CLI 型 |

## 使用前

**先确认没有现成 skill：**
```bash
npx skills find <关键词>
```
