---
title: Runtime State
aliases: [运行时状态, state management, agent state]
category: L1
created: 2026-04-06
updated: 2026-06-10
relations:
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[session-recovery]]"
    type: feeds
  - target: "[[multi-agent]]"
    type: depends_on
    evidence: "AgentScope Python 2.x 的 team worker 通过独立 AgentRecord/SessionRecord/AgentState 接入同一 MessageBus；multi-agent 状态隔离取决于 session/worker 粒度，而不是全局 ContextVar"
  - target: "[[session-recovery]]"
    type: supports
    evidence: "AgentScope 2.x 用 SessionRecord.state: AgentState 作为恢复边界，StorageBase 保存状态，ChatService 每轮重建 model/toolkit/workspace/middlewares，补充了 DeerFlow/LangGraph checkpoint 的另一种服务化恢复模型"
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
| 核心设计 | `AppStateStore`（统一状态树）+ `REPL.tsx`（交互运行时协调器）：前者承载 agent 工作台完整状态（权限/任务/MCP/IDE），后者串联 prompt 构建、query 驱动、工具审批 UI 和 hooks 生命周期 | 分为两层：`AppState`（31 字段 frozen dataclass，通过 `dataclasses.replace()` 不可变更新）+ `RuntimeBundle`（聚合所有活跃运行时对象的 dataclass，在调用栈中显式传递，替代隐式单例） | `ThreadState` TypedDict 在 LangGraph `AgentState` 基础上扩展工作区路径、产物、todos、上传文件等字段；持久化完全委托给 LangGraph checkpointer；虚拟路径抽象将 `/mnt/user-data/` 映射到 thread 数据目录，传入 `user_id` 时为 `{base_dir}/users/{user_id}/threads/{thread_id}/`，未传时才使用 legacy `{base_dir}/threads/{thread_id}/` | `SessionDB`（`hermes_state.py`）以单一 SQLite 文件（`~/.hermes/state.db`）作为唯一持久化后端，管理 session 元数据、完整消息历史、五维 token 用量和费用审计；WAL + FTS5 双轨，支持 gateway/CLI/worktree 多进程共享访问 | `AgentState` 是可持久化运行状态，`SessionRecord` 保存 workspace/model/team/source 等低频配置，model/toolkit/middleware/workspace 等活对象每轮由 `ChatService` 从 managers 重建；状态边界是“可恢复事实”，不是整个 agent 对象 |
| 关键特点 | REPL 不是薄渲染层，而是 prompt 构建参与者 + 工具审批 UI 持有者；工具审批 UI 与权限记录和 prompt 规则形成完整闭环；通过 async generator yield 机制解耦 REPL 与 query.ts | `RuntimeBundle` 显式传递，依赖关系透明可见，测试只需构造含 mock 对象的 bundle；frozen dataclass + `dataclasses.replace()` 使状态变更可追踪；`AppStateStore` 极简手写 40 行，无框架依赖 | 虚拟路径抽象：agent 无任何 thread-specific 路径硬编码；checkpointer 委托持久化：无需自实现序列化层；自定义 reducer（`merge_artifacts`/`merge_viewed_images`）实现语义化合并而非覆盖；`todos` 字段跨 turn 追踪任务进度 | 应用层抖动重试（20-150ms 随机，最多 15 次）替代 SQLite 内置 busy handler；Schema v1→v6 线性自动迁移；增量游标（`_last_flushed_db_idx`）防重复写入；压缩触发 session 链式分裂（`parent_session_id` + 标题自动续号）；守护线程异步标题生成 | `AgentState` 覆盖 summary/context/reply_id/cur_iter/permission/tool/task context；`ToolContext.activated_groups` 跟 session 走；`PermissionContext` 每轮注入 workspace workdir；`MessageBus` 把 inbox/wakeup/run lock 纳入 runtime 状态流 |
| 局限 | AppStateStore 边界不清，新功能容易随意挂载导致状态树膨胀；REPL.tsx 职责过重，局部修改影响面难以评估 | 无响应式/异步状态传播，状态变更需手动调用 `sync_app_state()`；手写 observable 缺乏错误隔离，单个订阅者抛异常可能影响其他订阅者 | State schema 固定（TypedDict 无法无侵入追加自定义字段）；无响应式状态传播；checkpointer 与文件系统副作用一致性边界模糊 | WAL 单写者上限：高并发写入 15 次重试（~2.25s）耗尽后仍可能抛 `database is locked`；`update_token_counts()` 的 CLI 增量/Gateway 绝对两种模式需调用方显式区分，误用易产生重复或丢失计数；session_id 格式无表级约束，外部传入非标准值不报错但会让前缀搜索语义失效 | `AgentState` 仍是粗粒度整体更新；runtime dependencies 依赖配置可重建，缺少版本化迁移会让旧 session 遇到工具/模型配置漂移；local workspace manager 当前只按 `basedir/agent_id` 隔离，Web 多租户需外层租户根目录和权限策略 |

## 设计权衡

### 方案对比

| 方案 | 适用场景 | 优势 | 劣势 | 代表实现 |
|------|---------|------|------|---------|
| **A. 全局单例** | 脚本、CLI 工具、单 agent | 随处访问，代码量少，上手快 | 测试困难（隐式依赖），多 agent 场景子 agent 互相污染 | 早期 agent 框架、简单脚本 |
| **B. 显式依赖注入（RuntimeBundle）** | 需要测试的系统、multi-agent | 依赖关系透明，mock 替换简单，无全局污染 | 调用栈越深 boilerplate 越多，参数传递链长 | OpenHarness `RuntimeBundle` |
| **C. 响应式状态（Reactive/Observable）** | 复杂 UI agent、实时状态同步 | 状态变更自动传播，UI 无需手动刷新，细粒度更新 | 学习曲线陡，调试难（谁触发了变更？），运行时开销 | Claude Code MobX/React observable |
| **D. SQLite 持久化状态库** | 多进程共享、需要计费审计、跨会话检索 | 原子性写入有保障、跨进程可见、FTS 全文检索开箱即用、无需自实现序列化 | WAL 单写者限制、并发写压力需应用层重试缓解、schema 演进需维护迁移脚本 | Hermes Agent `SessionDB` |

### 场景决策指南

- **Web 分布式 agent / 多用户 session 服务** → `AgentState + SessionRecord + runtime dependency rebuild`（AgentScope Python 2.x 方案）。只持久化 summary/context/reply/tool/task/permission 这些可恢复事实，model client、MCP session、workspace、middleware 每轮从配置和 managers 重建。好处是多进程友好，不需要让 agent 对象常驻；代价是工具组、model config、workspace manager 必须版本化，否则旧 session 会遇到配置漂移。

  代码证据：`src/agentscope/state/_state.py`、`app/storage/_model/_session.py`、`app/_service/_chat.py`。

- **同一 session 可能被 HTTP、team worker、schedule/background task 同时唤醒** → `MessageBus` 作为 runtime control plane。session lock 防止并发执行，event replay/live fan-out 支持前端重连，inbox/wakeup 让 idle session 可被异步事件拉起。agent-os 的本地形态可以用 in-process bus，Web 分布式形态再替换为 Redis/NATS。

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
7. **state 与可重建依赖没有版本边界**：AgentScope 2.x 的 `AgentState` 保存工具组名、permission/context/task 等事实，但 Toolkit/model/workspace 每轮重建。如果工具名、group 名、model config 或 workspace policy 升级后没有迁移策略，旧 session 可以被读出却无法完整继续。对策：给 session config 和 tool group schema 加版本号，恢复时做兼容转换或明确提示需要新 session。

8. **把活对象放进可恢复 state**：模型 client、MCP session、workspace 实例、文件句柄、middleware 都不应进入持久化 state。AgentScope 2.x 的做法是只保存 ID/config/state，恢复时重建活对象。对策：把 `State` 和 `RuntimeDependencies` 拆开，序列化测试只覆盖前者。

## 相关模式

- [[state-module-tree-serialization]] — 历史 AgentScope 1.x 模式；2.x 已迁移到 `AgentState + SessionRecord + MessageBus`

## L2 详情

- [[runtime-state--claude-code]]
- [[runtime-state--openharness]]
- [[runtime-state--deer-flow]]
- [[runtime-state--hermes-agent]]
- [[runtime-state--agentscope]]
