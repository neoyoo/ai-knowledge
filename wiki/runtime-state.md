---
title: Runtime State
aliases: [运行时状态, state management, agent state]
category: L1
created: 2026-04-06
updated: 2026-04-15
relations:
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[session-recovery]]"
    type: feeds
  - target: "[[multi-agent]]"
    type: depends_on
    evidence: "AgentScope ContextVar 隔离边界是 asyncio Task，多 agent 并发场景下 run_id/project 共享问题直接影响 multi-agent 状态隔离设计"
  - target: "[[session-recovery]]"
    type: supports
    evidence: "AgentScope StateModule 递归序列化 + SessionBase 三后端为 session-recovery 提供了框架无关的通用快照/恢复路径，补充了 DeerFlow/LangGraph checkpointer 绑定框架的另一极设计"
  - target: "[[sandbox-isolation]]"
    type: supports
    evidence: "持久 IPython kernel 模式（E2B/agentscope-runtime）要求沙箱会话生命周期绑定 runtime session，创建/暂停/销毁需要协调；runtime-state 的 session 边界决定了 sandbox-isolation 的资源归属粒度"
sources: [claude-code, openharness, deer-flow, hermes-agent, agentscope]
---

## 一句话定义

Agent 运行时的状态容器 — 管理当前会话信息、配置、运行模式和生命周期。

## 核心问题

- 哪些状态是全局的，哪些是单轮的？
- 状态怎么序列化（给 checkpoint/recovery 用）？
- 多 agent 场景下状态怎么隔离？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent | AgentScope |
|------|------------|-------------|----------|-------------|-----------|
| 核心设计 | `AppStateStore`（统一状态树）+ `REPL.tsx`（交互运行时协调器）：前者承载 agent 工作台完整状态（权限/任务/MCP/IDE），后者串联 prompt 构建、query 驱动、工具审批 UI 和 hooks 生命周期 | 分为两层：`AppState`（31 字段 frozen dataclass，通过 `dataclasses.replace()` 不可变更新）+ `RuntimeBundle`（聚合所有活跃运行时对象的 dataclass，在调用栈中显式传递，替代隐式单例） | `ThreadState` TypedDict 在 LangGraph `AgentState` 基础上扩展工作区路径、产物、todos、上传文件等字段；持久化完全委托给 LangGraph checkpointer；虚拟路径抽象（`/mnt/user-data/` → `threads/{thread_id}/`），agent 逻辑与存储布局解耦 | `SessionDB`（`hermes_state.py`）以单一 SQLite 文件（`~/.hermes/state.db`）作为唯一持久化后端，管理 session 元数据、完整消息历史、五维 token 用量和费用审计；WAL + FTS5 双轨，支持 gateway/CLI/worktree 多进程共享访问 | 三层正交设计：**进程级配置**（`_ConfigCls` + `ContextVar` — 多租户隔离）+ **模块级状态树**（`StateModule` 递归序列化 — 零侵入自动发现子模块）+ **会话持久化**（`SessionBase` 三后端适配：JSON 文件 / Redis / Alibaba Tablestore） |
| 关键特点 | REPL 不是薄渲染层，而是 prompt 构建参与者 + 工具审批 UI 持有者；工具审批 UI 与权限记录和 prompt 规则形成完整闭环；通过 async generator yield 机制解耦 REPL 与 query.ts | `RuntimeBundle` 显式传递，依赖关系透明可见，测试只需构造含 mock 对象的 bundle；frozen dataclass + `dataclasses.replace()` 使状态变更可追踪；`AppStateStore` 极简手写 40 行，无框架依赖 | 虚拟路径抽象：agent 无任何 thread-specific 路径硬编码；checkpointer 委托持久化：无需自实现序列化层；自定义 reducer（`merge_artifacts`/`merge_viewed_images`）实现语义化合并而非覆盖；`todos` 字段跨 turn 追踪任务进度 | 应用层抖动重试（20-150ms 随机，最多 15 次）替代 SQLite 内置 busy handler；Schema v1→v6 线性自动迁移；增量游标（`_last_flushed_db_idx`）防重复写入；压缩触发 session 链式分裂（`parent_session_id` + 标题自动续号）；守护线程异步标题生成 | `ContextVar` 驱动并发安全：不同 asyncio Task 无锁持有独立 `run_id`/配置，天然支持同进程多租户；`StateModule.__setattr__` 钩子零侵入自动发现子模块（赋值即注册）；`allow_not_exist=True` 宽松加载语义统一首次启动与状态恢复路径；Redis 后端通过 `GETEX` 原子操作实现 sliding TTL |
| 局限 | AppStateStore 边界不清，新功能容易随意挂载导致状态树膨胀；REPL.tsx 职责过重，局部修改影响面难以评估 | 无响应式/异步状态传播，状态变更需手动调用 `sync_app_state()`；手写 observable 缺乏错误隔离，单个订阅者抛异常可能影响其他订阅者 | State schema 固定（TypedDict 无法无侵入追加自定义字段）；无响应式状态传播；checkpointer 与文件系统副作用一致性边界模糊 | WAL 单写者上限：高并发写入 15 次重试（~2.25s）耗尽后仍可能抛 `database is locked`；`update_token_counts()` 的 CLI 增量/Gateway 绝对两种模式需调用方显式区分，误用易产生重复或丢失计数；session_id 格式无表级约束，外部传入非标准值不报错但会让前缀搜索语义失效 | 全量快照无增量 diff，长对话序列化开销线性增长；进程级配置以 `ContextVar` 隔离仅到 asyncio Task 粒度，同 Task 内多 agent 共享 `run_id`；`strict=True` 反序列化缺乏前向兼容 migration 机制；无 `on_save`/`on_load` 生命周期钩子，可观测性需上层封装；Tablestore 后端强依赖阿里云专有 SDK |

## 设计权衡

### 方案对比

| 方案 | 适用场景 | 优势 | 劣势 | 代表实现 |
|------|---------|------|------|---------|
| **A. 全局单例** | 脚本、CLI 工具、单 agent | 随处访问，代码量少，上手快 | 测试困难（隐式依赖），多 agent 场景子 agent 互相污染 | 早期 agent 框架、简单脚本 |
| **B. 显式依赖注入（RuntimeBundle）** | 需要测试的系统、multi-agent | 依赖关系透明，mock 替换简单，无全局污染 | 调用栈越深 boilerplate 越多，参数传递链长 | OpenHarness `RuntimeBundle` |
| **C. 响应式状态（Reactive/Observable）** | 复杂 UI agent、实时状态同步 | 状态变更自动传播，UI 无需手动刷新，细粒度更新 | 学习曲线陡，调试难（谁触发了变更？），运行时开销 | Claude Code MobX/React observable |
| **D. SQLite 持久化状态库** | 多进程共享、需要计费审计、跨会话检索 | 原子性写入有保障、跨进程可见、FTS 全文检索开箱即用、无需自实现序列化 | WAL 单写者限制、并发写压力需应用层重试缓解、schema 演进需维护迁移脚本 | Hermes Agent `SessionDB` |

### 场景决策指南

- **同进程多租户 / 多用户并发 Agent 服务**（每个用户请求在独立 asyncio Task 中运行）→ `ContextVar` 驱动的进程级配置隔离（AgentScope `_ConfigCls` 方案）。该模式无需显式传递 run_id/project，所有 Task 自动各持独立配置快照，锁竞争为零。注意隔离边界是 Task 而非 agent 实例——同一 Task 内多 agent 仍共享配置，需在应用层额外做 agent 粒度的 `run_id` 区分。

  代码证据：`src/agentscope/_run_config.py` — `_ConfigCls` 所有字段用 `ContextVar` 包裹；与 OpenHarness 显式 `RuntimeBundle` 的区别在于：前者适合高并发无状态服务（路由层创建 Task 即自动隔离），后者适合强类型依赖注入的测试友好场景。

- **Agent 状态树含嵌套子模块（记忆、工具状态等），需要一键序列化全量快照**（checkpoint / 热重启）→ `StateModule` 递归序列化树（AgentScope 方案）。继承 `StateModule` 后，子模块在 `__setattr__` 赋值时自动注册，调用 `state_dict()` 即可得到包含所有嵌套模块的完整 JSON 快照，无需手动处理嵌套。

  代码证据：`src/agentscope/module/_state_module.py`；相比 DeerFlow 委托 LangGraph checkpointer（与框架深度绑定）、Hermes Agent 自定义 SQLite 序列化（需维护 schema 迁移），AgentScope `StateModule` 方案框架无关、可移植，但代价是全量写入无增量 diff。适合会话规模可控（< 数千条消息）且需要灵活切换存储后端的场景。

- **需要跨后端（本地开发 / 生产 Redis / 云端 Tablestore）无缝切换会话持久化**→ `SessionBase` 三后端统一接口（AgentScope 方案）。三后端共享相同的 `save_session_state` / `load_session_state` 接口，切换后端只需替换 session 实例，上层 agent 代码零改动。Redis 后端额外支持 sliding TTL（`GETEX` 原子刷新），适合需要自动过期清理的生产环境。

  代码证据：`src/agentscope/session/_session_base.py`、`_json_session.py`、`_redis_session.py`、`_tablestore_session.py`。

- **脚本 / CLI / 单次任务** → 全局单例够用，不必过度设计。关键是在进入 multi-agent 之前识别这个边界。
- **需要单元测试的 agent 系统** → 显式依赖注入（RuntimeBundle 模式）。将所有运行时对象聚合为一个可构造的 bundle，测试时只需替换 bundle 中的 mock 对象，无需 patch 全局状态。
- **多 agent 并发场景** → 强制使用显式注入 + 不可变状态（frozen dataclass + `replace()` 模式）。每个子 agent 持有独立的状态快照，避免共享可变对象。
- **有实时 UI 的 agent（工具审批、进度显示）** → 响应式状态。但需同时维护不可变快照（用于调试和 replay），不能只依赖 observable。
- **多进程共享状态 + 需要跨会话检索或计费审计** → SQLite WAL 模式（SessionDB 方案）。FTS5 虚拟表让历史消息全文检索零成本，五维 token + 费用元数据从写入到落库形成完整审计链。注意 WAL 单写者上限，高并发写时必须配合应用层抖动重试，而非依赖 SQLite 内置 busy handler。

### 常见陷阱

1. **全局单例 + multi-agent**：子 agent 并发修改同一全局状态，导致竞态条件和状态污染。对策：进入 multi-agent 时必须切换为显式注入，每个子 agent 持有独立状态副本。
2. **响应式状态无 immutable snapshot**：调试时状态已被后续变更覆盖，无法还原出问题时的现场。对策：关键状态变更时记录 snapshot，或使用 append-only 事件日志（event sourcing）。
3. **状态过碎（100 个扁平字段）**：AppStateStore 随功能增加无边界挂载，最终成为"垃圾桶对象"。对策：按业务域划分子状态对象（权限域、任务域、MCP 域），每个子域有明确 owner。
4. **REPL / 协调器职责膨胀**：把 prompt 构建、工具审批、任务管理全塞进一个协调器，局部改动影响面难以评估。对策：认知域（query/推理）和交互域（UI/审批）必须分离，通过接口而非直接引用通信。
5. **SQLite busy handler 确定性退避 → 护送效应**：多进程同时写入时，SQLite 内置的等待节奏一致，所有进程可能以相同周期碰撞。对策：在应用层用随机抖动重试（如 20-150ms），配合 `BEGIN IMMEDIATE` 在事务开始即抢锁，不在 commit 时才暴露竞争。
6. **token 计数 CLI/Gateway 二元模式混用**：CLI 路径应用增量累加，Gateway 路径（每条消息新建 agent 实例）应用绝对覆盖；混用会产生重复计数或归零。对策：在 token 更新调用处明确标注 `absolute=True/False`，并在单元测试中同时覆盖两种路径。
7. **`strict=True` 加载旧版本快照失败**：agent 迭代新增 `register_state` 字段后，旧版本快照缺少该键，`load_state_dict(strict=True)` 直接抛异常。对策：版本迭代时将新增字段设为可选（`strict=False` + 显式默认值兜底），并维护快照格式版本号，上线前验证新旧版本快照的双向兼容性。（来源：AgentScope `StateModule`）

8. **全量快照 + 高频保存导致存储/序列化开销线性膨胀**：每次 `save_session_state` 序列化所有注册模块完整状态，无 diff/patch 机制。对策：长对话场景控制保存频率（如每 N 轮或仅在关键节点保存），或在应用层对记忆内容做压缩/截断后再触发快照。（来源：AgentScope `SessionBase`）

## L2 详情

- [[runtime-state--claude-code]]
- [[runtime-state--openharness]]
- [[runtime-state--deer-flow]]
- [[runtime-state--hermes-agent]]
- [[runtime-state--agentscope]]
