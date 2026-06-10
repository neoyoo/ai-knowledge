---
title: Tool System
aliases: [工具系统, tool dispatch, function calling, tool use]
category: L1
created: 2026-04-06
updated: 2026-06-10
relations:
  - target: "[[mcp-skills]]"
    type: extends
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[query-loop]]"
    type: depends_on
    evidence: "AgentScope Toolkit.call_tool() 返回 ToolChunk/ToolResponse 流，Agent reply 循环以 async for 消费；ResetTools 元工具通过 AgentState.tool_context.activated_groups 改变下一轮可见工具集，工具系统与 agent loop 之间存在运行时反馈"
  - target: "[[evaluation-observability]]"
    type: supports
    evidence: "AgentScope 2.x 通过 TracingMiddleware 包裹 reply/model-call/tool-execution 生命周期，工具调用会产生 OpenTelemetry GenAI span；可观测性作为 middleware 接入，而不是散落到业务工具中"
  - target: "[[sandbox-isolation]]"
    type: depends_on
    evidence: "代码执行类工具（run_python/bash/code_interpreter）的隔离强度决定了整个工具系统的安全边界上限；不同沙箱档次（subprocess/Docker/gVisor/Firecracker/WASM）支撑的威胁模型不同，直接约束 tool-system 可暴露给外部用户的工具集"
sources: [claude-code, openharness, deer-flow, hermes-agent, agentscope, tencentdb-agent-memory]
---

## 一句话定义

Agent 运行时中负责发现、选择、调用和管理外部工具的子系统。

## 核心问题

- 怎么让模型知道有哪些工具可用？
- 怎么把模型的意图转成实际的工具调用？
- 怎么处理权限和安全？
- 工具调用失败了怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent | AgentScope |
|------|------------|-------------|----------|-------------|-----------|
| 核心设计 | 完整的工具执行运行时（Tool Execution Runtime）：工具协议对象 + 分层工具池 + 批处理/流式双引擎，权限控制深度嵌入形成"模型可见性 + OS 级沙箱"双层防线 | 以 `BaseTool` 抽象类为核心，每个工具携带 Pydantic 输入模型，通过 `to_api_schema()` 自动生成 Anthropic 兼容 JSON Schema；MCP 工具通过 `McpToolAdapter` 全自动包装，注册表是普通 `dict[str, BaseTool]` | `get_available_tools()` 从配置工具、内置工具、MCP 扩展、ACP 跨框架 agent 四源聚合，引入延迟工具注册表（DeferredToolRegistry）：MCP 工具不暴露给模型，通过 `tool_search` 按需发现，从根本上解决大规模工具集的 token 膨胀问题 | 模块级单例注册表（`ToolRegistry`）+ 工具文件自注册模式：每个 `tools/*.py` 在 import 时主动写入注册表，`check_fn + requires_env` 实现运行时可用性过滤；Toolset 系统声明式分组，Distribution 系统为 RL/批量训练提供概率采样 | 以 `ToolBase`、`ToolGroup`、`Toolkit` 和 `AgentState.tool_context` 为核心；workspace 内置工具、MCP、skills、planning、team、schedule/background tools 都汇入 Toolkit；工具组激活状态随 session state 持久化，LLM 可通过 `ResetTools` 动态切换可见工具组 |
| 关键特点 | Fail-closed 并发安全（`isConcurrencySafe` 默认串行）；流式预启动（模型生成时提前启动工具）；三层工具池分离"存在/可见/可执行"三个边界<br>**注意**：并发安全性取决于**工具本身**，而非框架。AgentScope 2.x 也显式区分 `is_concurrency_safe`，agent 会把安全工具并发分批执行、把 `Bash`/`Write`/`Edit` 等非安全工具串行分批执行。实践建议：无副作用的工具允许并发；有文件/DB 写入的工具显式标记串行。 | Pydantic `input_model` 兼顾参数验证与 API Schema 生成，一套定义两用；`McpToolAdapter` 通过 `create_model()` 实现 MCP 工具零手工接入；工具自我声明只读性（`is_read_only()`），权限逻辑下沉到工具层 | 延迟工具注册表彻底解决 MCP token 膨胀；护栏独立于调度；ACP 跨框架调用；tool output externalization 默认把大 `ToolMessage` 落文件并保留 preview/ref/read_file 指引；slash skill 单轮激活把用户显式选择的 skill context 注入隐藏 HumanMessage 并写入 RunJournal | 三层结果防溢出（工具内截断 → 单结果持久化 → Turn 预算聚合）；三路 async bridge 保护持久化 HTTP 客户端生命周期；Schema 动态修补防止 model 幻觉调用不存在的工具；`_AGENT_LOOP_TOOLS` 在注册表与分发层之间制造显式拆分点 | `ToolBase` 把 permission/read-only/concurrency/state injection/rule suggestion 做成工具协议；`ToolGroup` 同时管理 tools、MCP clients 和 skill instructions；`Toolkit.call_tool()` 统一产出 `ToolChunk` stream + 最终 `ToolResponse`；`ResetTools.input_schema` 按当前 ToolGroup 动态生成 |
| 局限 | MCP 工具的并发安全声明依赖外部 server，可靠性低于内建工具；自动审批分类器决策逻辑对用户不可见 | 无工具分组或上下文条件过滤，所有工具始终全量暴露，工具数量增多后增大模型上下文压力；无工具调用重试或降级策略 | 护栏能力基础（规则型，不支持语义级策略）；tool_search 增加 RTT（延迟发现需额外一轮工具调用）；工具来源配置分散；外置文件引用解决当前 context 预算，但不是长期 evidence graph；slash skill 是用户显式激活，不解决自动 skill selection | `_AGENT_LOOP_TOOLS` 硬编码导致注册与分发语义必须在两处同步维护；Distribution 采样不验证 check_fn，可能采到实际不可用的 toolset；async bridge timeout 硬编码 300s，无法按工具粒度覆盖 | Toolkit 本身不持久化，session 恢复后需由 workspace/app managers 重建相同工具来源；dynamic tool group 状态只保存激活组名，若部署时工具组配置漂移会出现旧上下文中的工具名不可解析；MCP stateful client 与 workspace 生命周期绑定，分布式部署需明确连接归属 |

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|---------|---------|---------|
| 方案 A：静态注册 | 所有工具在启动时注册完毕，运行时工具集固定不变 | 工具集稳定的内部工具、编程助手、CLI agent | Claude Code、OpenHarness |
| 方案 B：动态发现（MCP/Plugin） | 工具通过协议在运行时发现和注册，支持第三方扩展 | 开放平台、工具市场、多租户 SaaS | MCP 生态、OpenAI Plugin（已废弃）、各类插件市场 |
| 方案 C：模型自选（Adaptive） | 把大量工具全量暴露给模型，由模型按上下文自行选择 | 工具库庞大但每次任务只用其中一小部分 | 部分 RAG + 工具混合系统 |
| 方案 D：运行时过滤（check_fn） | 工具静态注册，但每次返回 schema 前执行可用性检查，过滤掉当前环境不满足条件的工具 | 多租户/多平台 agent（不同部署环境工具集不同）、依赖外部 API key 的工具 | Hermes Agent |

**权限控制子方案对比：**

| 权限模式 | 适用阶段 | 风险敞口 |
|---------|---------|---------|
| 全开放（无审批） | 本地开发、原型验证 | 高，任何工具随意执行 |
| 三级制（auto/ask/deny） | 日常生产使用 | 中，敏感操作需人工确认 |
| 沙箱隔离（容器/虚拟化） | 高安全生产环境 | 低，但运维成本高 |

### 场景决策指南

**如果你的工具集固定、内部自用 → 选方案 A（静态注册）+ 三级权限制**
- 原因：工具集稳定时静态注册最可预测——启动时校验一次，运行时零动态开销，出问题好定位
- 注意：权限控制不能省——开发时全开放没问题，上生产前必须加上 auto/ask/deny 分级，否则一次 prompt injection 就能触发危险操作

**如果你在构建开放平台、让第三方接入工具 → 选方案 B（MCP/Plugin）+ 沙箱隔离**
- 原因：第三方工具质量无法保证，动态发现机制必须配合沙箱运行，不能让外部工具直接访问主进程资源
- 注意：每个动态工具都是潜在的安全攻击面，需要对工具 schema 做验证（防止恶意描述操控模型行为）；动态加载有延迟，对响应时间敏感的场景要做好预加载或缓存

**如果工具很多但每次任务只用几个 → 选方案 A 静态注册 + 按任务过滤（不要选方案 C）**
- 原因：把 100+ 工具全量传给模型会占用大量 token 预算（工具描述可能占 2000-5000 tokens），且模型在工具太多时选择准确率下降；按任务类型预过滤到 5-15 个相关工具效果最好
- 注意：过滤逻辑要简单可靠，用规则而非用另一个 LLM 来判断"应该用哪些工具"——否则引入了额外的失败点

**如果工具集因部署环境而变化（API key 可选、多平台、多租户）→ 选方案 D（运行时 check_fn 过滤）**
- 原因：工具注册逻辑无需分叉，统一在启动时全量注册，check_fn 在每次 `get_definitions()` 时按当前环境过滤——同一套代码库，Telegram 部署和 CLI 部署自动得到不同的工具集
- 注意：check_fn 缓存至关重要——同一 toolset 下多个工具共享同一 check_fn 时，缓存保证同一次调用只执行一次环境检查，否则高频调用会反复触发 I/O（如 env var 检查、网络探活）；check_fn 抛出异常应视为不可用而非报错，防止单个工具的配置错误传播到调度层

**如果工具集在运行时需要动态切换（不同任务阶段需要不同工具子集）→ 选方案 E（ToolGroup + 元工具自管理，AgentScope 模式）**
- 原因：静态注册方案（方案 A）在启动后工具集固定；方案 D 的 check_fn 是环境驱动的过滤，不是 LLM 意图驱动的切换。AgentScope 的 `reset_tools` 是一个普通工具，LLM 可以在任务执行过程中主动调用它来激活/停用 ToolGroup，实现工具集的自适应管理。
- 代码证据：`Toolkit.get_tool_schemas()` 返回当前可见工具 schema；`ResetTools.input_schema` property 会基于当前 ToolGroup 动态 `create_model()`，参数内容与当前分组配置同步——LLM 拿到的字段反映真实组结构，不会出现幻觉调用不存在组名的情况。
- 与已有源对比：DeerFlow 的 `tool_search` 解决"工具太多时降低 token 消耗"问题，侧重发现；AgentScope 的元工具方案解决"任务阶段不同时工具集应动态变化"问题，侧重切换。两者场景正交，不互斥。
- 注意：`basic` 组工具永远激活不可被 LLM 关闭，安全关键工具应注册到 `basic` 组；AgentScope 2.x 的激活状态在 `AgentState.tool_context.activated_groups`，但工具组定义本身仍由每轮重建的 Toolkit 提供，分布式多 Worker 必须保证工具配置一致。

**如果工具需要内部上下文但不应暴露给 LLM（如 agent state、workspace/session 上下文）→ state injection / runtime wrapper**
- 原因：把内部上下文做成 LLM 参数会污染 schema，也会让模型误以为自己能控制这些字段。AgentScope 2.x 的做法是工具声明 `is_state_injected=True`，`Toolkit.call_tool()` 执行时额外注入 `_agent_state`；这类参数不出现在 tool input schema 中。
- 代码证据：`ToolBase.is_state_injected` 声明工具是否需要 `_agent_state`；`Toolkit.call_tool()` 在非 MCP/非 external tool 上把当前 `AgentState` 注入 kwargs。
- 注意：API key、tenant id、session id 等不应依赖模型输入，建议从 workspace/session config、permission context 或专门 storage 读取；如果只是闭包 bind，也要确保 schema 与执行参数分离。

**如果需要为 RL 训练或批量评测构建多样化工具环境 → Distribution 采样（Hermes 模式）**
- 原因：固定工具集训练出的模型对未见工具组合泛化性差；独立概率采样（非互斥）让每个 batch 样本拥有不同的工具组合，模型接触到更丰富的工具情境
- 注意：Distribution 采样应在工具有效性验证之后执行（即采样后仍需 check_fn 确认可用），否则无效工具集会污染训练数据；保底逻辑（至少选一个）是防止空工具集导致训练样本无效的关键安全网

**如果工具结果很大且后续必须可追溯 → evidence store + recall tools（TencentDB Agent Memory 模式）**
- 原因：单纯截断工具结果会省 token，但后续无法回到原文；单纯把完整结果留在 messages 会撑爆 context。TencentDB Agent Memory 先把完整工具结果写入 `refs/*.md`，再把 summary/ref/node_id 注入上下文，并提供 memory/conversation search 与 `read_file` 做按需展开。
- 注意：这会增加 recall 类工具面。auto-recall 本身是在 hook 内查询 store，不是模型 function call；但当注入片段不足时，LLM 可能额外调用搜索/读取工具。生产实现需要 per-turn recall budget，不能只依赖工具描述里的“最多 3 次”。

**如果只是当前 run 的工具输出太大，但不需要长期语义图 → tool output externalization（DeerFlow 模式）**
- 原因：DeerFlow 把超大 `ToolMessage` 外置到 outputs 目录，message 中只留 head/tail preview、完整路径和 `read_file` 指引；历史 `ToolMessage` 进入 model call 前也会被预算 middleware 处理。这比 evidence store 简单，适合保护当前上下文，不承担长期 memory。
- 注意：外置路径必须绑定 workspace/session 权限边界，不能把任意宿主文件路径暴露给远程 channel。分布式 Web 形态需要把本地路径替换为 object store URI + 权限校验。

### 常见陷阱

- **暴露 100+ 工具给模型**：工具描述占满 token 预算，模型选择准确率下降，响应变慢变贵——实测超过 20 个工具后准确率开始明显下滑，超过 40 个基本失控
- **没有权限控制就上生产**：一次精心构造的 prompt 就能触发文件删除、代码执行或数据外泄，三级权限（auto/ask/deny）是最低门槛，不是可选项
- **工具描述写太长**：每个工具描述超过 200 字就是在白白烧 token，好的工具描述应该是"动词+对象+关键限制"，20-50 字足够
- **动态工具不做 schema 验证**：MCP 工具的 schema 描述来自外部，恶意构造的工具描述可以误导模型调用其他工具，必须在注册时做格式校验和内容审查；Hermes 的 schema 动态修补（`execute_code` 工具列表 + `browser_navigate` 描述）展示了更细粒度的防护：确保工具描述中提到的工具名与实际可用列表完全一致，消灭幻觉调用的根源
- **忘记处理工具调用失败**：工具报错后没有 retry/fallback 策略，模型会困惑或重复调用——需要明确定义"错误信息怎么回传给模型"以及"最多重试几次"
- **不处理工具结果溢出**：工具返回几十 KB 甚至几百 KB 的内容时，单轮对话的 context 会被快速撑满——Hermes 的三层防溢出（工具内截断 → 单结果持久化到文件 → Turn 级预算聚合）是工程上可借鉴的完整方案；最小实践是给每个工具设置输出字符上限，溢出时改为"摘要 + 完整内容路径"。**各家实现差异**：Claude Code 未文档化统一策略，由工具自行处理；AgentScope 通过 `convert_tool_result_to_string()` 截断后写本地文件（单机场景适用，分布式需改造）；Hermes Agent 实施三层防护——工具内部截断 → 失败时持久化文件 → turn 级别总预算上限。选择建议：单机 agent 用 AgentScope 模式即可；生产环境或分布式部署推荐 Hermes 三层模式。
- **只截断不保留 evidence**：工具结果被截断后，如果没有 `result_ref` / evidence URI，LLM 后续只能重跑工具或猜测细节。TencentDB Agent Memory 展示了另一条路线：完整结果进 evidence store，context 只保留摘要和可展开 handle。
- **recall 工具没有硬预算**：memory/conversation/read 类工具是必要补充，但会把上下文压力转成 function calling 压力。工具层应按 turn 记录调用次数和总返回 token，超过预算时拒绝或降级返回。
- **注册与分发语义分裂**：当部分工具需要外部状态（如 session store、memory store）而另一部分不需要时，分发层必须有明确的拆分边界——Hermes 的 `_AGENT_LOOP_TOOLS` 硬编码方案能用，但随工具增加维护成本线性上升；更好的做法是在 `ToolEntry` 上声明依赖类型，让注册表本身成为唯一真相来源
- **在事件循环内直接用 `asyncio.run()` 执行异步工具**：会触发"Cannot run a new event loop while already running"错误，并导致绑定旧 loop 的 HTTP 客户端在 GC 时报"Event loop is closed"——需要根据调用上下文（主线程/工作线程/async gateway）维护不同策略的持久化 loop，Hermes 的三路 async bridge 是此问题的完整解决方案
- **把内部上下文暴露成模型参数**：为让工具拿到 API key / session / agent state，把这些字段放进 schema 让 LLM 传入，会造成安全和一致性问题。更好的做法是在执行层注入内部上下文，或让工具从 permission/session/workspace storage 中读取，确保 schema 只包含模型应控制的业务参数。
- **不区分 MCP 有状态/无状态场景就选同一客户端**：browser-use 等需要跨调用保持上下文的 MCP 服务必须用有状态连接，简单 API 封装类服务可以每次短连接。AgentScope 2.x 已合并为统一 `MCPClient`，用 `mcp_config` + `is_stateful` 表达组合；STDIO 强制 stateful，HTTP 可按服务特性选择。

## 下游依赖说明

**prompt-system 依赖 tool-system**：prompt/context 预算策略与工具调用结构强耦合。`tool_use` 和 `tool_result` 必须成对存在，provider formatter 必须把工具序列转成合法 API 消息。AgentScope 2.x 已不再用 formatter 做 `_truncate()` 自动截断；当前边界是 `FormatterBase._group_messages()` 维护工具序列合法性，`Agent.compress_context()` 和 `model.count_tokens()` 负责上下文压缩触发。**影响面**：若 tool-system 改变结果结构（如增加嵌套层级）或消息 role 约定，formatter、context 压缩和多模态 tool result 投影需同步更新。

**context-management 依赖 tool-system**：上下文预算计算必须感知工具边界。截断和压缩均依赖 `tool_call_ids` 集合确保不产生孤儿消息；工具调用密度直接影响 `keep_recent` 的实际 token 保留量（一轮大量 tool_use/result 可轻易占满）；token 估算还必须纳入 tools schema 序列化开销（50+ 工具额外消耗 20-30K token）。**影响面**：若 tool-system 改变结果大小策略（如输出截断上限、文件持久化阈值）或 schema 体积，context 的压缩触发阈值和 token 估算逻辑需联动调整。

**memory-system 依赖 tool-system**：长期记忆不是纯后台索引，往往要暴露为可调用工具。TencentDB Agent Memory 的 `tdai_memory_search` / `tdai_conversation_search` 说明：auto-recall 可以先低成本注入导航信息，但真正展开原文仍要靠工具系统承接权限、预算、返回大小和调用次数控制。**影响面**：若工具层缺少 per-turn budget 或 evidence URI 权限隔离，memory 召回会变成新的成本和安全入口。

## L2 详情

- [[tool-system--claude-code]]
- [[tool-system--openharness]]
- [[tool-system--deer-flow]]
- [[tool-system--hermes-agent]]
- [[tool-system--agentscope]]
