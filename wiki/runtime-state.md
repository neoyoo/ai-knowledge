---
title: Runtime State
aliases: [运行时状态, state management, agent state]
category: L1
created: 2026-04-06
updated: 2026-04-06
relations:
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[session-recovery]]"
    type: feeds
sources: [claude-code, openharness, deer-flow]
---

## 一句话定义

Agent 运行时的状态容器 — 管理当前会话信息、配置、运行模式和生命周期。

## 核心问题

- 哪些状态是全局的，哪些是单轮的？
- 状态怎么序列化（给 checkpoint/recovery 用）？
- 多 agent 场景下状态怎么隔离？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow |
|------|------------|-------------|----------|
| 核心设计 | `AppStateStore`（统一状态树）+ `REPL.tsx`（交互运行时协调器）：前者承载 agent 工作台完整状态（权限/任务/MCP/IDE），后者串联 prompt 构建、query 驱动、工具审批 UI 和 hooks 生命周期 | 分为两层：`AppState`（31 字段 frozen dataclass，通过 `dataclasses.replace()` 不可变更新）+ `RuntimeBundle`（聚合所有活跃运行时对象的 dataclass，在调用栈中显式传递，替代隐式单例） | `ThreadState` TypedDict 在 LangGraph `AgentState` 基础上扩展工作区路径、产物、todos、上传文件等字段；持久化完全委托给 LangGraph checkpointer；虚拟路径抽象（`/mnt/user-data/` → `threads/{thread_id}/`），agent 逻辑与存储布局解耦 |
| 关键特点 | REPL 不是薄渲染层，而是 prompt 构建参与者 + 工具审批 UI 持有者；工具审批 UI 与权限记录和 prompt 规则形成完整闭环；通过 async generator yield 机制解耦 REPL 与 query.ts | `RuntimeBundle` 显式传递，依赖关系透明可见，测试只需构造含 mock 对象的 bundle；frozen dataclass + `dataclasses.replace()` 使状态变更可追踪；`AppStateStore` 极简手写 40 行，无框架依赖 | 虚拟路径抽象：agent 无任何 thread-specific 路径硬编码；checkpointer 委托持久化：无需自实现序列化层；自定义 reducer（`merge_artifacts`/`merge_viewed_images`）实现语义化合并而非覆盖；`todos` 字段跨 turn 追踪任务进度 |
| 局限 | AppStateStore 边界不清，新功能容易随意挂载导致状态树膨胀；REPL.tsx 职责过重，局部修改影响面难以评估 | 无响应式/异步状态传播，状态变更需手动调用 `sync_app_state()`；手写 observable 缺乏错误隔离，单个订阅者抛异常可能影响其他订阅者 | State schema 固定（TypedDict 无法无侵入追加自定义字段）；无响应式状态传播；checkpointer 与文件系统副作用一致性边界模糊 |

## 设计权衡

### 方案对比

| 方案 | 适用场景 | 优势 | 劣势 | 代表实现 |
|------|---------|------|------|---------|
| **A. 全局单例** | 脚本、CLI 工具、单 agent | 随处访问，代码量少，上手快 | 测试困难（隐式依赖），多 agent 场景子 agent 互相污染 | 早期 agent 框架、简单脚本 |
| **B. 显式依赖注入（RuntimeBundle）** | 需要测试的系统、multi-agent | 依赖关系透明，mock 替换简单，无全局污染 | 调用栈越深 boilerplate 越多，参数传递链长 | OpenHarness `RuntimeBundle` |
| **C. 响应式状态（Reactive/Observable）** | 复杂 UI agent、实时状态同步 | 状态变更自动传播，UI 无需手动刷新，细粒度更新 | 学习曲线陡，调试难（谁触发了变更？），运行时开销 | Claude Code MobX/React observable |

### 场景决策指南

- **脚本 / CLI / 单次任务** → 全局单例够用，不必过度设计。关键是在进入 multi-agent 之前识别这个边界。
- **需要单元测试的 agent 系统** → 显式依赖注入（RuntimeBundle 模式）。将所有运行时对象聚合为一个可构造的 bundle，测试时只需替换 bundle 中的 mock 对象，无需 patch 全局状态。
- **多 agent 并发场景** → 强制使用显式注入 + 不可变状态（frozen dataclass + `replace()` 模式）。每个子 agent 持有独立的状态快照，避免共享可变对象。
- **有实时 UI 的 agent（工具审批、进度显示）** → 响应式状态。但需同时维护不可变快照（用于调试和 replay），不能只依赖 observable。

### 常见陷阱

1. **全局单例 + multi-agent**：子 agent 并发修改同一全局状态，导致竞态条件和状态污染。对策：进入 multi-agent 时必须切换为显式注入，每个子 agent 持有独立状态副本。
2. **响应式状态无 immutable snapshot**：调试时状态已被后续变更覆盖，无法还原出问题时的现场。对策：关键状态变更时记录 snapshot，或使用 append-only 事件日志（event sourcing）。
3. **状态过碎（100 个扁平字段）**：AppStateStore 随功能增加无边界挂载，最终成为"垃圾桶对象"。对策：按业务域划分子状态对象（权限域、任务域、MCP 域），每个子域有明确 owner。
4. **REPL / 协调器职责膨胀**：把 prompt 构建、工具审批、任务管理全塞进一个协调器，局部改动影响面难以评估。对策：认知域（query/推理）和交互域（UI/审批）必须分离，通过接口而非直接引用通信。

## L2 详情

- [[runtime-state--claude-code]]
- [[runtime-state--openharness]]
- [[runtime-state--deer-flow]]
