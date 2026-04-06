---
title: "Runtime State — OpenHarness"
category: L2
parent: "[[runtime-state]]"
source: openharness
source_version: "0.1.0"
confidence: high
created: 2026-04-06
updated: 2026-04-06
---

## 概述
OpenHarness 的运行时状态分为两层：`AppState` 是 31 字段的冻结 dataclass，通过 `dataclasses.replace()` 实现不可变更新，由手写的 40 行 `AppStateStore` 提供 subscribe/set 发布-订阅能力；`RuntimeBundle` 则是一个聚合所有活跃运行时对象的 dataclass，在整个调用栈中显式传递，替代隐式单例模式。

## 架构分析
### AppState：不可变配置快照
`AppState` 是 31 字段的 frozen dataclass，涵盖 model、mode、theme、provider、vim、voice、fast_mode、effort、passes、mcp_connected 等所有可配置项。状态更新通过 `dataclasses.replace()` 产生新实例，旧实例保持不变，天然线程安全且易于比对变更。

### AppStateStore：手写观察者
`AppStateStore` 是 40 行的手写 observable store，提供 `subscribe()` 注册监听回调、`set()` 推送新状态两个核心操作。极简实现，无外部依赖，适合 TUI 渲染层监听配置变更触发重绘。

### RuntimeBundle：显式依赖容器
`RuntimeBundle` 将所有活跃运行时对象打包为单一 dataclass：`api_client`、`mcp_manager`、`tool_registry`、`app_state`、`hook_executor`、`engine`、`commands`。整个调用栈通过参数显式传递 `RuntimeBundle`，而非依赖全局单例或 DI 容器。每次交互后调用 `sync_app_state()` 读取当前设置与 MCP 状态，推送新 `AppState`。

### 关键代码路径
- `openharness/state/app_state.py` — `AppState` frozen dataclass 定义，31 个字段
- `openharness/state/store.py` — `AppStateStore` 手写 observable store，40 行
- `openharness/ui/runtime.py` — `RuntimeBundle` 定义与 `sync_app_state()` 实现

## 设计亮点
- `RuntimeBundle` 作为单一 dataclass 在调用栈中显式传递，对比 Claude Code 的隐式单例模式，依赖关系透明可见，测试时只需构造含 mock 对象的 `RuntimeBundle` 即可
- frozen dataclass + `dataclasses.replace()` 的不可变更新模式，使状态变更可追踪、可回滚，调试友好
- `AppStateStore` 极简手写实现，40 行内完成发布-订阅，无框架依赖，维护成本低

## 局限性
- 无响应式/异步状态传播：状态变更不会自动触发下游更新，需要手动调用 `sync_app_state()`
- 手写 observable 缺乏错误隔离：单个订阅者抛异常可能影响其他订阅者的通知
- `RuntimeBundle` 字段增多后，构造成本和传递噪音会随系统规模线性增长

## 来源
- 源码版本：OpenHarness 0.1.0 (HKUDS/OpenHarness)
- 分析深度：源码级
