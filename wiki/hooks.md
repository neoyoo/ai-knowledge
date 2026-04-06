---
title: Hooks
aliases: [钩子, event hooks, lifecycle hooks]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: extends
sources: [claude-code]
---

## 一句话定义

事件驱动的扩展点 — 在 agent 运行的特定时机插入自定义逻辑。

## 核心问题

- 哪些生命周期事件值得暴露为 hook？
- Hook 执行失败会阻塞主流程吗？
- 怎么保证 hook 的执行顺序？
- 用户怎么注册和管理 hook？

## 各家对比

| 维度 | Claude Code |
|------|------------|
| 核心设计 | 完整类型化事件系统（28+ 事件类型）+ shell 子进程执行引擎 + 双向 JSON 通信协议，来源支持用户配置/Skills/Plugins 三路合并注册 |
| 关键特点 | Hook 可做权限决策（allow/deny）且不能绕过 settings.json deny 规则；异步 hook 模式不阻塞主循环；原子性 plugin hook 热重载（clear-then-register 解决卸载后 ghost hook 问题） |
| 局限 | Hook 本质是 shell 子进程，隔离性依赖 shell 安全；matcher 只匹配工具名，不支持对工具参数的条件匹配 |

## 设计权衡

- **强类型协议 vs 自由文本 callback**：hook 响应通过 Zod schema 严格验证，compile-time 断言保证 SDK 类型与 schema 同步，把协议一致性保障前移到编译期而非依赖运行时检查。
- **Hook allow 不能绕过 settings deny**：权限批准的最终决策权在 settings.json 规则，hook 只能简化用户交互（跳过弹窗），不能提升权限——安全底线由配置而非扩展点控制。

## L2 详情

- [[hooks--claude-code]]
