# Tools Cookbook

> Agent 工具系统的实操指南。工具设计原理见 [[wiki/tool-system]]。

本板块的工具定义来自三个经过市场验证的开源项目源码分析：
- **Claude Code** — Anthropic 官方 CLI（42 tools，TypeScript + Zod）
- **DeerFlow** — 多 Agent 研究框架（18 tools，Python + LangChain）
- **OpenHarness** — 开源 Agent 运行时（35+ tools，Python + Pydantic）

## 子板块

### [Tool Definitions](definitions/_index.md)
按功能分类的工具定义，包含可复制的 schema、跨项目对比、最佳实践。

### [Tool Patterns](patterns/_index.md)
工具编排和设计模式：工具选择策略、并行调用、错误处理、权限模型。

## 怎么用

1. 需要某类工具 → 去 definitions/ 找对应类别
2. 每个工具有三个项目的实现对比 → 选最适合你场景的
3. Tool schema 可直接复制到你的 agent 项目中
4. 工具编排问题 → 去 patterns/ 找设计模式
