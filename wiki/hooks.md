---
title: Hooks
aliases: [钩子, event hooks, lifecycle hooks]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: extends
sources: [claude-code, openharness]
---

## 一句话定义

事件驱动的扩展点 — 在 agent 运行的特定时机插入自定义逻辑。

## 核心问题

- 哪些生命周期事件值得暴露为 hook？
- Hook 执行失败会阻塞主流程吗？
- 怎么保证 hook 的执行顺序？
- 用户怎么注册和管理 hook？

## 各家对比

| 维度 | Claude Code | OpenHarness |
|------|------------|-------------|
| 核心设计 | 完整类型化事件系统（28+ 事件类型）+ shell 子进程执行引擎 + 双向 JSON 通信协议，来源支持用户配置/Skills/Plugins 三路合并注册 | 4 种事件类型（SESSION_START/END、PRE/POST_TOOL_USE）+ 4 种 hook 类型（command、http、prompt、agent），通过 `settings.json` 或插件 `hooks.json` 声明；`HookReloader` mtime 轮询实现热重载 |
| 关键特点 | Hook 可做权限决策（allow/deny）且不能绕过 settings.json deny 规则；异步 hook 模式不阻塞主循环；原子性 plugin hook 热重载（clear-then-register 解决卸载后 ghost hook 问题） | LLM 评估型 hook（`prompt`/`agent` 类型）允许用自然语言描述策略条件，无需编写脚本即可实现语义级 policy enforcement；mtime 热重载使 hook 调试无需重启进程；`block_on_failure: true` 可让 PRE_TOOL_USE hook 失败时阻断工具执行 |
| 局限 | Hook 本质是 shell 子进程，隔离性依赖 shell 安全；matcher 只匹配工具名，不支持对工具参数的条件匹配 | 仅 4 种事件，相比 Claude Code 28+ 事件，可干预的节点非常有限；缺乏类似 Zod 的运行时 schema 校验；所有 hook 串行执行，无并发调度 |

## 设计权衡

- **强类型协议 vs 自由文本 callback**：hook 响应通过 Zod schema 严格验证，compile-time 断言保证 SDK 类型与 schema 同步，把协议一致性保障前移到编译期而非依赖运行时检查。
- **Hook allow 不能绕过 settings deny**：权限批准的最终决策权在 settings.json 规则，hook 只能简化用户交互（跳过弹窗），不能提升权限——安全底线由配置而非扩展点控制。

## L2 详情

- [[hooks--claude-code]]
- [[hooks--openharness]]
