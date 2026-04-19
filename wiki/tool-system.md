---
title: Tool System
aliases: [工具系统, tool dispatch, function calling, tool use]
category: L1
created: 2026-04-06
updated: 2026-04-15
relations:
  - target: "[[mcp-skills]]"
    type: extends
  - target: "[[query-loop]]"
    type: feeds
  - target: "[[query-loop]]"
    type: depends_on
    evidence: "AgentScope Toolkit 的 call_tool_function 是 AsyncGenerator，Agent reply 循环以 async for 消费——工具系统的流式执行接口直接耦合 agent loop 的迭代模式；元工具 reset_equipped_tools 在 loop 内部由 LLM 调用改变下一轮可见工具集，工具系统与 agent loop 之间存在运行时反馈"
  - target: "[[evaluation-observability]]"
    type: supports
    evidence: "AgentScope Toolkit 的执行路径最外层为 @trace_toolkit（OpenTelemetry span），所有工具调用自动产生可观测性 span，无需上层 agent 代码显式埋点；这是同类项目中唯一把 OTEL 追踪作为工具系统内置层的实现"
  - target: "[[sandbox-isolation]]"
    type: depends_on
    evidence: "代码执行类工具（run_python/bash/code_interpreter）的隔离强度决定了整个工具系统的安全边界上限；不同沙箱档次（subprocess/Docker/gVisor/Firecracker/WASM）支撑的威胁模型不同，直接约束 tool-system 可暴露给外部用户的工具集"
sources: [claude-code, openharness, deer-flow, hermes-agent, agentscope]
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
| 核心设计 | 完整的工具执行运行时（Tool Execution Runtime）：工具协议对象 + 分层工具池 + 批处理/流式双引擎，权限控制深度嵌入形成"模型可见性 + OS 级沙箱"双层防线 | 以 `BaseTool` 抽象类为核心，每个工具携带 Pydantic 输入模型，通过 `to_api_schema()` 自动生成 Anthropic 兼容 JSON Schema；MCP 工具通过 `McpToolAdapter` 全自动包装，注册表是普通 `dict[str, BaseTool]` | `get_available_tools()` 从配置工具、内置工具、MCP 扩展、ACP 跨框架 agent 四源聚合，引入延迟工具注册表（DeferredToolRegistry）：MCP 工具不暴露给模型，通过 `tool_search` 按需发现，从根本上解决大规模工具集的 token 膨胀问题 | 模块级单例注册表（`ToolRegistry`）+ 工具文件自注册模式：每个 `tools/*.py` 在 import 时主动写入注册表，`check_fn + requires_env` 实现运行时可用性过滤；Toolset 系统声明式分组，Distribution 系统为 RL/批量训练提供概率采样 | 以 `Toolkit`（StateModule）为核心，统一管理三类来源：Python 函数、MCP 客户端（有状态/无状态双模式）、Agent Skill 目录。工具以「ToolGroup」为粒度做动态启用/停用，执行层全部统一为 `AsyncGenerator[ToolResponse, None]` 流式接口，叠加洋葱式 middleware + OpenTelemetry 追踪；独特设计：LLM 可通过内置元工具 `reset_equipped_tools` 在运行时自主切换自身可见工具集 |
| 关键特点 | Fail-closed 并发安全（`isConcurrencySafe` 默认串行）；流式预启动（模型生成时提前启动工具）；三层工具池分离"存在/可见/可执行"三个边界<br>**注意**：并发安全性取决于**工具本身**，而非框架。AgentScope 全程异步并发也安全运行，因为其工具函数无共享副作用。`isConcurrencySafe` 的价值在于让工具**显式声明**安全性，而非强制串行化。实践建议：无副作用的工具默认允许并发；有文件/DB 写入的工具显式标记串行。 | Pydantic `input_model` 兼顾参数验证与 API Schema 生成，一套定义两用；`McpToolAdapter` 通过 `create_model()` 实现 MCP 工具零手工接入；工具自我声明只读性（`is_read_only()`），权限逻辑下沉到工具层 | 延迟工具注册表彻底解决 MCP token 膨胀，是同类项目中最优雅的解法；护栏独立于调度（GuardrailMiddleware 是正交安全层，可独立替换）；ACP 跨框架调用（将其他框架 agent 注册为本地工具） | 三层结果防溢出（工具内截断 → 单结果持久化 → Turn 预算聚合）；三路 async bridge 保护持久化 HTTP 客户端生命周期；Schema 动态修补防止 model 幻觉调用不存在的工具；`_AGENT_LOOP_TOOLS` 在注册表与分发层之间制造显式拆分点 | `preset_kwargs` 实现工具参数隐藏（API key/session_id 等预设参数对 LLM 完全不可见，执行时自动合并）；`extended_model` 动态扩展 schema（运行时给已注册工具附加额外参数，无需修改原函数签名）；JSON schema 从 docstring 自动提取（Google/Numpy 风格），MCP 工具直接从 inputSchema 提取；`async_execution=True` 后台异步执行长时工具，立即返回 task_id |
| 局限 | MCP 工具的并发安全声明依赖外部 server，可靠性低于内建工具；自动审批分类器决策逻辑对用户不可见 | 无工具分组或上下文条件过滤，所有工具始终全量暴露，工具数量增多后增大模型上下文压力；无工具调用重试或降级策略 | 护栏能力基础（规则型，不支持语义级策略）；tool_search 增加 RTT（延迟发现需额外一轮工具调用）；工具来源配置分散于两个文件 | `_AGENT_LOOP_TOOLS` 硬编码导致注册与分发语义必须在两处同步维护；Distribution 采样不验证 check_fn，可能采到实际不可用的 toolset；async bridge timeout 硬编码 300s，无法按工具粒度覆盖 | 工具组激活状态为进程内单例，跨 session 恢复需重新注册所有工具；`preset_kwargs` 不支持动态修改（修改需 remove + 重新注册）；工具 schema 完全依赖 docstring 格式，无 fallback；有状态 MCP 客户端多实例需手动 LIFO 顺序关闭（底层 SDK 约束透传）；Agent Skill 无版本管理 |

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
- 原因：静态注册方案（方案 A）在启动后工具集固定；方案 D 的 check_fn 是环境驱动的过滤，不是 LLM 意图驱动的切换。AgentScope 的 `reset_equipped_tools` 是一个被注册为普通工具的 Python 方法，LLM 可以在任务执行过程中主动调用它来激活/停用 ToolGroup，实现工具集的自适应管理。
- 代码证据：`Toolkit.get_json_schemas()` 每次调用时动态注入 `reset_equipped_tools` 的参数（通过 `extended_model`），参数内容与当前分组配置完全同步——LLM 拿到的参数字段永远反映真实的组结构，不会出现幻觉调用不存在组名的情况。
- 与已有源对比：DeerFlow 的 `tool_search` 解决"工具太多时降低 token 消耗"问题，侧重发现；AgentScope 的元工具方案解决"任务阶段不同时工具集应动态变化"问题，侧重切换。两者场景正交，不互斥。
- 注意：`basic` 组工具永远激活不可被 LLM 关闭，安全关键工具应注册到 `basic` 组；工具组激活状态是进程内单例，分布式多 Worker 场景下需额外同步机制。

**如果工具函数需要隐藏部分参数不暴露给 LLM（如 API key、session_id、内部上下文）→ preset_kwargs 模式（AgentScope）**
- 原因：常见做法是为每个工具包装一个 closure，把敏感参数 bind 进去。AgentScope 的 `preset_kwargs` 在注册时声明，框架自动从 JSON schema 中剔除这些字段，执行时再合并——无需为每个工具写 wrapper，注册代码更干净。
- 代码证据：`call_tool_function` 执行时：`kwargs = {**tool_func.preset_kwargs, **tool_call["input"]}`；`_parse_tool_function` 在构造 JSON schema 时会跳过 preset_kwargs 中已有的参数名。
- 与已有源对比：OpenHarness 和 Hermes 均需要手动在工具类/函数外部包装来隐藏内部参数；AgentScope 是目前分析的源中唯一把「参数隐藏」做成一等公民注册选项的实现。
- 注意：`preset_kwargs` 不支持运行时动态修改（无 `update_preset_kwargs` 接口），若内部状态需要频繁变化，应通过其他机制（如在工具函数内部从共享 StateModule 读取）而非 preset_kwargs。

**如果需要为 RL 训练或批量评测构建多样化工具环境 → Distribution 采样（Hermes 模式）**
- 原因：固定工具集训练出的模型对未见工具组合泛化性差；独立概率采样（非互斥）让每个 batch 样本拥有不同的工具组合，模型接触到更丰富的工具情境
- 注意：Distribution 采样应在工具有效性验证之后执行（即采样后仍需 check_fn 确认可用），否则无效工具集会污染训练数据；保底逻辑（至少选一个）是防止空工具集导致训练样本无效的关键安全网

### 常见陷阱

- **暴露 100+ 工具给模型**：工具描述占满 token 预算，模型选择准确率下降，响应变慢变贵——实测超过 20 个工具后准确率开始明显下滑，超过 40 个基本失控
- **没有权限控制就上生产**：一次精心构造的 prompt 就能触发文件删除、代码执行或数据外泄，三级权限（auto/ask/deny）是最低门槛，不是可选项
- **工具描述写太长**：每个工具描述超过 200 字就是在白白烧 token，好的工具描述应该是"动词+对象+关键限制"，20-50 字足够
- **动态工具不做 schema 验证**：MCP 工具的 schema 描述来自外部，恶意构造的工具描述可以误导模型调用其他工具，必须在注册时做格式校验和内容审查；Hermes 的 schema 动态修补（`execute_code` 工具列表 + `browser_navigate` 描述）展示了更细粒度的防护：确保工具描述中提到的工具名与实际可用列表完全一致，消灭幻觉调用的根源
- **忘记处理工具调用失败**：工具报错后没有 retry/fallback 策略，模型会困惑或重复调用——需要明确定义"错误信息怎么回传给模型"以及"最多重试几次"
- **不处理工具结果溢出**：工具返回几十 KB 甚至几百 KB 的内容时，单轮对话的 context 会被快速撑满——Hermes 的三层防溢出（工具内截断 → 单结果持久化到文件 → Turn 级预算聚合）是工程上可借鉴的完整方案；最小实践是给每个工具设置输出字符上限，溢出时改为"摘要 + 完整内容路径"。**各家实现差异**：Claude Code 未文档化统一策略，由工具自行处理；AgentScope 通过 `convert_tool_result_to_string()` 截断后写本地文件（单机场景适用，分布式需改造）；Hermes Agent 实施三层防护——工具内部截断 → 失败时持久化文件 → turn 级别总预算上限。选择建议：单机 agent 用 AgentScope 模式即可；生产环境或分布式部署推荐 Hermes 三层模式。
- **注册与分发语义分裂**：当部分工具需要外部状态（如 session store、memory store）而另一部分不需要时，分发层必须有明确的拆分边界——Hermes 的 `_AGENT_LOOP_TOOLS` 硬编码方案能用，但随工具增加维护成本线性上升；更好的做法是在 `ToolEntry` 上声明依赖类型，让注册表本身成为唯一真相来源
- **在事件循环内直接用 `asyncio.run()` 执行异步工具**：会触发"Cannot run a new event loop while already running"错误，并导致绑定旧 loop 的 HTTP 客户端在 GC 时报"Event loop is closed"——需要根据调用上下文（主线程/工作线程/async gateway）维护不同策略的持久化 loop，Hermes 的三路 async bridge 是此问题的完整解决方案
- **工具参数用 closure 而非声明式隐藏**：为隐藏 API key / session 上下文，给每个工具写一个专门的 closure wrapper，代码量随工具数线性增长。更好的做法是用类似 AgentScope `preset_kwargs` 的声明式方案，框架层负责 schema 剔除和执行时合并，注册代码保持整洁。
- **不区分 MCP 有状态/无状态场景就选同一客户端**：browser-use 等需要跨调用保持上下文的 MCP 服务必须用有状态客户端（维持长连接 session），简单 API 封装类服务用无状态客户端即可（每次新建 session，无连接泄漏风险）。混用会导致有状态服务状态丢失或无状态服务资源浪费。AgentScope 的双模式客户端（`HttpStatelessClient` vs `StdIOStatefulClient`）以相同接口覆盖两类场景，是值得借鉴的设计。

## 下游依赖说明

**prompt-system 依赖 tool-system**：prompt 截断策略与工具调用结构强耦合。`tool_use` 和 `tool_result` 必须成对存在——截断时若只删其中一条，provider API 会直接报 400。AgentScope `_truncate()` 用 `tool_call_ids` 集合追踪配对，以完整对为最小删除单元；`promote_tool_result_images` 还需感知工具返回的内容类型决定是否提升图片。**影响面**：若 tool-system 改变结果结构（如增加嵌套层级）或消息 role 约定，prompt 的截断逻辑和图片处理路径需同步更新。

**context-management 依赖 tool-system**：上下文预算计算必须感知工具边界。截断和压缩均依赖 `tool_call_ids` 集合确保不产生孤儿消息；工具调用密度直接影响 `keep_recent` 的实际 token 保留量（一轮大量 tool_use/result 可轻易占满）；token 估算还必须纳入 tools schema 序列化开销（50+ 工具额外消耗 20-30K token）。**影响面**：若 tool-system 改变结果大小策略（如输出截断上限、文件持久化阈值）或 schema 体积，context 的压缩触发阈值和 token 估算逻辑需联动调整。

## L2 详情

- [[tool-system--claude-code]]
- [[tool-system--openharness]]
- [[tool-system--deer-flow]]
- [[tool-system--hermes-agent]]
- [[tool-system--agentscope]]
