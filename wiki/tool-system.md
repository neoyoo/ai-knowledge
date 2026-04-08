---
title: Tool System
aliases: [工具系统, tool dispatch, function calling, tool use]
category: L1
created: 2026-04-06
updated: 2026-04-08
relations:
  - target: "[[mcp-skills]]"
    type: extends
  - target: "[[query-loop]]"
    type: feeds
sources: [claude-code, openharness, deer-flow, hermes-agent]
---

## 一句话定义

Agent 运行时中负责发现、选择、调用和管理外部工具的子系统。

## 核心问题

- 怎么让模型知道有哪些工具可用？
- 怎么把模型的意图转成实际的工具调用？
- 怎么处理权限和安全？
- 工具调用失败了怎么处理？

## 各家对比

| 维度 | Claude Code | OpenHarness | DeerFlow | Hermes Agent |
|------|------------|-------------|----------|-------------|
| 核心设计 | 完整的工具执行运行时（Tool Execution Runtime）：工具协议对象 + 分层工具池 + 批处理/流式双引擎，权限控制深度嵌入形成"模型可见性 + OS 级沙箱"双层防线 | 以 `BaseTool` 抽象类为核心，每个工具携带 Pydantic 输入模型，通过 `to_api_schema()` 自动生成 Anthropic 兼容 JSON Schema；MCP 工具通过 `McpToolAdapter` 全自动包装，注册表是普通 `dict[str, BaseTool]` | `get_available_tools()` 从配置工具、内置工具、MCP 扩展、ACP 跨框架 agent 四源聚合，引入延迟工具注册表（DeferredToolRegistry）：MCP 工具不暴露给模型，通过 `tool_search` 按需发现，从根本上解决大规模工具集的 token 膨胀问题 | 模块级单例注册表（`ToolRegistry`）+ 工具文件自注册模式：每个 `tools/*.py` 在 import 时主动写入注册表，`check_fn + requires_env` 实现运行时可用性过滤；Toolset 系统声明式分组，Distribution 系统为 RL/批量训练提供概率采样 |
| 关键特点 | Fail-closed 并发安全（`isConcurrencySafe` 默认串行）；流式预启动（模型生成时提前启动工具）；三层工具池分离"存在/可见/可执行"三个边界 | Pydantic `input_model` 兼顾参数验证与 API Schema 生成，一套定义两用；`McpToolAdapter` 通过 `create_model()` 实现 MCP 工具零手工接入；工具自我声明只读性（`is_read_only()`），权限逻辑下沉到工具层 | 延迟工具注册表彻底解决 MCP token 膨胀，是同类项目中最优雅的解法；护栏独立于调度（GuardrailMiddleware 是正交安全层，可独立替换）；ACP 跨框架调用（将其他框架 agent 注册为本地工具） | 三层结果防溢出（工具内截断 → 单结果持久化 → Turn 预算聚合）；三路 async bridge 保护持久化 HTTP 客户端生命周期；Schema 动态修补防止 model 幻觉调用不存在的工具；`_AGENT_LOOP_TOOLS` 在注册表与分发层之间制造显式拆分点 |
| 局限 | MCP 工具的并发安全声明依赖外部 server，可靠性低于内建工具；自动审批分类器决策逻辑对用户不可见 | 无工具分组或上下文条件过滤，所有工具始终全量暴露，工具数量增多后增大模型上下文压力；无工具调用重试或降级策略 | 护栏能力基础（规则型，不支持语义级策略）；tool_search 增加 RTT（延迟发现需额外一轮工具调用）；工具来源配置分散于两个文件 | `_AGENT_LOOP_TOOLS` 硬编码导致注册与分发语义必须在两处同步维护；Distribution 采样不验证 check_fn，可能采到实际不可用的 toolset；async bridge timeout 硬编码 300s，无法按工具粒度覆盖 |

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

**如果需要为 RL 训练或批量评测构建多样化工具环境 → Distribution 采样（Hermes 模式）**
- 原因：固定工具集训练出的模型对未见工具组合泛化性差；独立概率采样（非互斥）让每个 batch 样本拥有不同的工具组合，模型接触到更丰富的工具情境
- 注意：Distribution 采样应在工具有效性验证之后执行（即采样后仍需 check_fn 确认可用），否则无效工具集会污染训练数据；保底逻辑（至少选一个）是防止空工具集导致训练样本无效的关键安全网

### 常见陷阱

- **暴露 100+ 工具给模型**：工具描述占满 token 预算，模型选择准确率下降，响应变慢变贵——实测超过 20 个工具后准确率开始明显下滑，超过 40 个基本失控
- **没有权限控制就上生产**：一次精心构造的 prompt 就能触发文件删除、代码执行或数据外泄，三级权限（auto/ask/deny）是最低门槛，不是可选项
- **工具描述写太长**：每个工具描述超过 200 字就是在白白烧 token，好的工具描述应该是"动词+对象+关键限制"，20-50 字足够
- **动态工具不做 schema 验证**：MCP 工具的 schema 描述来自外部，恶意构造的工具描述可以误导模型调用其他工具，必须在注册时做格式校验和内容审查；Hermes 的 schema 动态修补（`execute_code` 工具列表 + `browser_navigate` 描述）展示了更细粒度的防护：确保工具描述中提到的工具名与实际可用列表完全一致，消灭幻觉调用的根源
- **忘记处理工具调用失败**：工具报错后没有 retry/fallback 策略，模型会困惑或重复调用——需要明确定义"错误信息怎么回传给模型"以及"最多重试几次"
- **不处理工具结果溢出**：工具返回几十 KB 甚至几百 KB 的内容时，单轮对话的 context 会被快速撑满——Hermes 的三层防溢出（工具内截断 → 单结果持久化到文件 → Turn 级预算聚合）是工程上可借鉴的完整方案；最小实践是给每个工具设置输出字符上限，溢出时改为"摘要 + 完整内容路径"
- **注册与分发语义分裂**：当部分工具需要外部状态（如 session store、memory store）而另一部分不需要时，分发层必须有明确的拆分边界——Hermes 的 `_AGENT_LOOP_TOOLS` 硬编码方案能用，但随工具增加维护成本线性上升；更好的做法是在 `ToolEntry` 上声明依赖类型，让注册表本身成为唯一真相来源
- **在事件循环内直接用 `asyncio.run()` 执行异步工具**：会触发"Cannot run a new event loop while already running"错误，并导致绑定旧 loop 的 HTTP 客户端在 GC 时报"Event loop is closed"——需要根据调用上下文（主线程/工作线程/async gateway）维护不同策略的持久化 loop，Hermes 的三路 async bridge 是此问题的完整解决方案

## L2 详情

- [[tool-system--claude-code]]
- [[tool-system--openharness]]
- [[tool-system--deer-flow]]
- [[tool-system--hermes-agent]]
