---
title: Hooks
aliases: [钩子, event hooks, lifecycle hooks]
category: L1
created: 2026-04-06
updated: 2026-04-08
relations:
  - target: "[[query-loop]]"
    type: extends
sources: [claude-code, openharness, deer-flow, hermes-agent]
---

## 一句话定义

事件驱动的扩展点 — 在 agent 运行的特定时机插入自定义逻辑。

## 核心问题

- 哪些生命周期事件值得暴露为 hook？
- Hook 执行失败会阻塞主流程吗？
- 怎么保证 hook 的执行顺序？
- 用户怎么注册和管理 hook？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|-------------|
| 核心设计 | 完整类型化事件系统（28+ 事件类型）+ shell 子进程执行引擎 + 双向 JSON 通信协议，来源支持用户配置/Skills/Plugins 三路合并注册 | 4 种事件类型（SESSION_START/END、PRE/POST_TOOL_USE）+ 4 种 hook 类型（command、http、prompt、agent），通过 `settings.json` 或插件 `hooks.json` 声明；`HookReloader` mtime 轮询实现热重载 | 无传统 hooks，以 middleware pipeline 作为扩展机制：每个 middleware 类暴露 `before_model`/`after_model`/`before_agent`/`after_agent`/`wrap_tool_call` 五个切入点，功能覆盖 hooks 所有典型场景且可访问完整 agent 状态 | **双轨并行**：`AIAgent` 上 9 个 Callback 字段（面向平台集成层，由 callsite 构建时注入）+ `PluginManager` 的 10 个具名 hook 事件（面向第三方插件，全局单例 `register_hook()` 注册）；两套系统在同一主循环内分开调用、不互通 |
| 关键特点 | Hook 可做权限决策（allow/deny）且不能绕过 settings.json deny 规则；异步 hook 模式不阻塞主循环；原子性 plugin hook 热重载（clear-then-register 解决卸载后 ghost hook 问题） | LLM 评估型 hook（`prompt`/`agent` 类型）允许用自然语言描述策略条件，无需编写脚本即可实现语义级 policy enforcement；mtime 热重载使 hook 调试无需重启进程；`block_on_failure: true` 可让 PRE_TOOL_USE hook 失败时阻断工具执行 | `@Next/@Prev` 装饰器无侵入插入（自定义 middleware 可精确指定位置，不修改框架源码）；GuardrailMiddleware 作为独立策略引擎（安全策略与 middleware 机制解耦）；Python 类实现类型安全，IDE 可做静态检查 | 回调全为纯 Python callable，零跨进程开销；`stream_delta_callback(None)` 以 `None` 值携带"轮次边界"带外语义；`pre_llm_call` 插件 hook 的返回值注入 user message（而非 system prompt），刻意保护 prefix cache 命中率；gateway 以"agent 实例缓存 + 每消息动态绑定 callback"解耦跨轮次状态 |
| 局限 | Hook 本质是 shell 子进程，隔离性依赖 shell 安全；matcher 只匹配工具名，不支持对工具参数的条件匹配 | 仅 4 种事件，相比 Claude Code 28+ 事件，可干预的节点非常有限；缺乏类似 Zod 的运行时 schema 校验；所有 hook 串行执行，无并发调度 | 无 shell-command hooks（集成外部自动化必须写 Python middleware）；无热重载（新 middleware 需重启服务）；无内置 LLM 评估策略（GuardrailProvider 无内置 LLM 评估器） | callback 参数无类型约束（签名不匹配运行时静默失败）；`tool_progress_callback` 多路复用 `event_type` 字符串，callsite 各自过滤，理解成本高；Plugin hook 的 `pre_tool_call`/`post_tool_call` 在工具执行路径中未找到实际调用点，疑似占位定义尚未落地；RL 环境（`HermesAgentLoop`）不复用 callback 架构，两套代码分叉维护 |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 无 Hook（硬编码） | 扩展点写死在代码里，无运行时配置 | 内部工具、行为固定、团队自己维护代码 | 简单脚本 agent |
| Shell Hook（命令行） | 事件触发时执行 shell 命令，用 JSON 做双向通信 | 开发者平台、CI/CD 集成、用户自定义自动化 | Claude Code |
| 函数 Hook（代码回调） | 注册语言原生函数作为事件处理器 | SDK/框架、高性能场景、需要类型安全的库 | LangChain callbacks |
| LLM 评估 Hook | Hook 条件是一个 LLM prompt，自然语言描述策略 | 语义级策略执行、难以用代码表达的模糊规则 | OpenHarness prompt/agent hook |
| 双轨 Hook（平台集成 + 插件扩展） | 两套系统分别面向不同消费者：平台 callback 由 callsite 构建时注入（携带 UI/通道上下文），插件 hook 通过全局注册由第三方扩展 | 多接入面平台（CLI/API/消息通道共用一个 agent 核心）、既需要平台定制又需要第三方插件的场景 | Hermes Agent |

### 场景决策指南

**如果是内部工具、行为完全固定 → 无 Hook**
扩展点是有成本的——每个 hook 都是一个间接层，一个潜在的故障点。如果行为不需要被外部定制，直接写死在代码里最简单可靠。

**如果是开发者平台、需要用户可自定义行为 → Shell Hook**
Shell 命令是最通用的接口：任意语言都能接入，行为对用户透明（shell 命令可读），和现有工具链（git hooks、CI 脚本）天然兼容。Claude Code 的 28+ 事件类型 + JSON 双向通信是这个方向的完整实现。

**如果是 SDK/框架，需要高性能和类型安全 → 函数 Hook**
没有进程 fork 开销，类型系统在编译期捕获错误。代价是语言锁定——Python SDK 的 hook 只能用 Python 写。

**如果需要执行难以用代码表达的模糊安全策略 → LLM 评估 Hook**
典型场景："这个文件操作是否涉及敏感数据？"。OpenHarness 的 `prompt`/`agent` hook 类型允许用自然语言描述策略，处理代码写不清楚的边界情况。只在真正需要语义判断时使用——每次 hook 触发都是一次 API 调用。

**如果是多接入面平台（CLI + API Gateway + 消息通道 + RL 训练）→ 双轨 Hook**
平台集成层（UI 渲染、流式展示、通道推送）和第三方插件扩展是两类完全不同的关切，混在一套系统里会导致 callsite 代码臃肿。Hermes 的解法是：平台 callback 由各个入口按需注入（CLI 注入全套 TUI 回调，Gateway 每消息动态绑定，RL 环境一个都不用），插件 hook 全局注册一次、到处生效。两套系统不互通，也不需要互通——各自的消费者不同。需要注意：Plugin hook 必须验证每个 hook 名确实有对应的调用点，否则容易出现"声明了但从未被触发"的占位定义。

### 常见陷阱

- **Hook 太多，每个 tool call 都触发**：每个 shell hook 都是一次进程 fork，串行执行时延迟叠加。在热路径（频繁触发的 tool call）上的 hook 必须做性能测试，考虑异步模式。
- **Hook 能修改请求和响应**：可写 hook 让系统行为变得不可预测——同样的输入可能产生不同输出，因为某个 hook 悄悄改了内容。调试时永远不知道是 agent 的问题还是 hook 改了数据。除非有充分理由，hook 应该只读。
- **Shell hook 没有超时设置**：一个卡住的网络请求或死锁的脚本会阻塞整个 agent 主循环。所有 shell hook 必须设置超时，且超时行为需要明确定义（是 fail-open 还是 fail-closed）。
- **安全 hook 能被其他 hook 绕过**：如果 hook 执行顺序不确定，一个 allow hook 可能在 deny hook 之前执行并缓存结果。Claude Code 的解法是 settings.json deny 规则具有最终否决权，hook 不能提升权限——安全底线必须在 hook 体系之外。
- **多路复用 callback 的 event_type 过滤陷阱**：把多个独立事件合并进单一 callback（如 Hermes 的 `tool_progress_callback`）可以减少参数数量，但每个 callsite 都要自行用 `if event_type != "xxx": return` 过滤，当 event_type 新增时所有 callsite 都可能受影响且不会有编译期提示。如非必要，分开的事件回调更自描述、更易维护。
- **Plugin hook 的占位定义陷阱**：声明了 `VALID_HOOKS` 集合，但不代表每个 hook 名都真的有对应的 `invoke_hook()` 调用点。Hermes 的 `pre_tool_call`/`post_tool_call` 可能从未被触发。设计 hook 体系时，每个 hook 名都必须有端到端的验证：注册 → 调用 → 效果。
- **Plugin hook 注入 context 走 user message 而非 system prompt**：若希望 plugin hook 注入的上下文在多轮对话中稳定生效，要注意 Hermes `pre_llm_call` 的设计：注入走 user message（每轮临时），而非 system prompt（全局持久）。这是刻意保护 prefix cache 的权衡——如果真的需要持久上下文，必须在 agent 构建时写入 system prompt，而不是依赖 hook 动态注入。

## L2 详情

- [[hooks--claude-code]]
- [[hooks--openharness]]
- [[hooks--deer-flow]]
- [[hooks--hermes-agent]]
