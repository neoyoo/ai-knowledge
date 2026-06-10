---
title: Agent Registry & Discovery
aliases: [Agent 注册发现, agent registry, service discovery for agents, AgentCard]
category: L1
created: 2026-04-19
updated: 2026-06-10
relations:
  - target: "[[multi-agent]]"
    type: feeds
    evidence: "多 agent 协作的前置条件——没有注册发现就只能硬编码点对点布线，N² 连接问题"
  - target: "[[channel-remote]]"
    type: depends_on
    evidence: "跨进程/跨 node agent 调用必须先知道对端在哪、能干什么"
  - target: "[[runtime-state]]"
    type: uses
    evidence: "注册信息（AgentCard、健康状态、心跳）本身是一种 runtime state，需要存储方案"
  - target: "[[tool-system]]"
    type: alternative
    evidence: "agent 能力声明和 tool 能力声明是同一抽象的两个层次——tool registry 解决'有哪些工具'，agent registry 解决'有哪些 agent'"
sources: [agentscope-java, openharness, claude-code, deer-flow, hermes-agent]
---

## 一句话定义

让 Agent 作为可独立寻址的实体，彼此"知道对方是谁、在哪、能干什么"的机制——Agent OS 的地址总线。

## 核心问题

- Agent 怎么被命名和寻址？硬编码地址 `http://worker-1:8080`，还是符号名 `webfetch-agent`？
- 新 agent 上线/下线时其他 agent 怎么知道？
- 多个相同能力的 agent 实例怎么选（负载均衡、能力匹配、session 亲和性）？
- 一个 agent 的能力如何声明，让别人按需发现（"谁能处理 OCR"→ 返回候选）？
- 心跳、健康检查、版本协商怎么做？

## Java 后端视角类比

从传统服务发现经验切入最直观。但 agent 领域有独特的三个差异，不能原样照搬。

| Java 后端 | Agent 领域对应 | 对应物 |
|---|---|---|
| Eureka / Nacos / Consul | AgentScope Java Nacos + AgentCard | 服务注册中心 |
| 服务接口签名（SOAP / gRPC proto） | AgentCard 能力声明（skills 字段） | 接口契约 |
| DNS / k8s Service | k8s Service 本身就是轻量 agent registry | 名字解析 + 负载均衡 |
| HTTP / gRPC 调用 | A2A 协议 | 调用协议 |
| Sentinel / Hystrix 熔断 | Token budget / rate limit | 保护层 |

**三个关键差异**：

1. **能力是语义描述而非精确接口签名**。Java 服务 `UserService.getUser(id)` 精确到方法签名；Agent 声明"我能处理 URL 页面提取、输出结构化数据"——发现时可能做"能力匹配"（按意图查）而不是名字查找
2. **调用是秒级 / 分钟级**（LLM 响应慢）。传统服务发现配合 ms 级 LB 策略；agent 需要超时、重试、队列削峰、熔断策略全重新设计
3. **Agent 可能有状态**（session 亲和性、prompt cache 保留）——不能简单 round-robin。同一用户的请求尽量路由到同一 worker，否则 prompt cache 失效、token 费用翻倍

## 各家对比

| 维度 | AgentScope Java | OpenHarness | Claude Code | DeerFlow | Hermes Agent |
|------|-----------|-------------|-------------|----------|-------------|
| **核心设计** | AgentCard + A2A JSON-RPC/SSE + Well-Known/Nacos resolver + Spring Boot starter，形成 Java 企业服务化链路；server ready 后注册 AgentCard，client 端 resolver 可订阅/缓存 | `TeamRecord` 内存字典，主 agent 启动子进程时记录 `(agent_id, process, pipe)`，进程间通信走 stdin/stdout | `swarm/backends/registry.ts` 只注册 swarm **后端类型**（本地/tmux/远程），不是注册 agent 实例 | `_background_tasks` dict 存运行中的 subagent 任务——本质是运行时 task 表，非跨进程 agent 注册 | `_active_children` 线程安全列表跟踪当前活跃的子 agent，用于 `interrupt()` 级联传播 |
| **存储形态** | AgentCard 可通过 well-known endpoint 发布，也可 release 到 Nacos；agent state 另由 `(userId, sessionId)` keyed store 持久化 | Python dict（进程内） | TypeScript Map（进程内） | Python dict（进程内） | Python list + Lock（进程内） |
| **发现协议** | `AgentCardResolver` 抽象，WellKnown resolver 处理 URL 发现，Nacos resolver 处理动态订阅缓存；server 侧 `AgentRegistry` 负责注册 | 无发现协议——硬编码+字典查询 | 无发现协议——swarm backend 通过配置选择 | 无 | 无 |
| **跨进程能力** | ✅ A2A JSON-RPC/SSE；AgentCard 作为跨进程能力契约 | 🟡 子进程 stdin/stdout，UTF-8 文本行协议 | ❌ 同进程 | ❌ 同进程（线程池） | ❌ 同进程（线程池） |
| **能力声明** | ✅ AgentCard 结构化字段（name、description、skills、capabilities、version） | ❌ 无形式化能力描述 | 🟡 子 agent 分 subagent_type，但能力是 prompt 隐含 | 🟡 subagent_type 配置，能力是 prompt 隐含 | 🟡 子 agent 工具通过 `frozenset(DELEGATE_BLOCKED_TOOLS)` 剥除；无正向能力声明 |
| **心跳 / 健康检查** | 🟡 Nacos 层可承接健康/动态注册；具体 agent 负载治理仍需上层策略 | 🟡 `broken pipe` 检测 + 子进程重启 | ❌ | ❌ | ❌ |
| **版本协商** | 🟡 AgentCard 暴露 version；Nacos resolver 可承接版本治理，具体调度策略需上层实现 | ❌ | ❌ | ❌ | ❌ |
| **生产成熟度** | ✅ Java/Spring/Nacos 生态路径完整，但对 Python/轻量部署偏重 | 内存状态，进程重启后团队关系丢失 | 单机 coordinator，作者自己承认"缺乏真正分布式协调" | 提及 Postgres 多实例部署但未展开 | 单机，Cron 无法水平扩展 |
| **关键文件** | `ReActAgent.java`、`AgentStateStore.java`、`AgentRegistry.java`、`AgentScopeA2aServer.java`、`NacosAgentCardResolver.java`、`NacosA2aRegistry.java`、`AgentscopeA2aAutoConfiguration.java` | `TeamRecord`（`multi-agent--openharness.md` L22-48） | `src/utils/swarm/backends/registry.ts` | `deerflow/tools/builtins/task_tool.py` `_background_tasks` | `hermes/agent/react_agent.py` `_active_children` |

### 关键观察

- **只有 AgentScope Java 做出了接近生产级的 agent registry 参考**。其余 4 家都是"运行时内存登记"，本质是为了跟踪当前生命周期内的子 agent，而不是让 agent 彼此"发现"
- **AgentCard 是真正的 agent 世界 API schema**。单独这一个抽象就把 agent 从"LLM 包装的黑盒"升级为"有契约的可组合单元"
- **Nacos 在这里的选择不是偶然**：阿里 Java/Spring 基础设施经验直接映射到 agent 服务化——但 Python AgentScope 2.x 当前主线是 session/workspace/team runtime，不应把 A2A/Nacos 能力继续归到 Python 仓库

## 设计权衡

### 方案对比

| 方案 | 核心机制 | 适合场景 | 成本 | 代表项目 |
|------|---------|---------|------|---------|
| **硬编码地址** | 代码写死 `http://subagent-svc:8080` | 单机或 subagent 类型固定且少 | 零 | 早期所有项目 |
| **k8s Service / Ingress** | 依赖容器编排自带的 DNS + LB + health check | 横向扩展 **无状态** subagent worker；k8s 基建已有 | 低（只要你已在 k8s） | 任何容器化部署 |
| **自建 Postgres/Redis 注册表** | 应用层表：worker_id / agent_type / capabilities / heartbeat / endpoint | 需要能力匹配、动态 worker 池、session 亲和性路由 | 中（~50 行代码 + 心跳维护逻辑） | trip-os 推荐 P2 路径 |
| **AgentCard + Nacos**（AgentScope Java 完整方案） | 标准化 AgentCard schema + Nacos 服务注册 + 版本锁定 | 公司已有 Nacos、需要多 agent 跨系统互操作、阿里云生态 | 高（多一个中间件） | AgentScope Java |
| **AgentCard + A2A**（协议标准化） | AgentCard + A2A JSON-RPC/SSE + resolver 策略 | 跨组织 agent 互联（别家的 agent 调你的） | 最高（协议学习 + resolver 部署） | AgentScope Java |

### 场景决策指南

**Subagent 类型固定 + 部署单一 → 硬编码 + k8s Service 组合**。一个 k8s Service DNS 名字（如 `webfetch.default.svc.cluster.local`）就是最小可用 registry，自带负载均衡 + 健康检查。**P1 阶段不需要更复杂的东西。**

**Subagent 动态 + 需要能力匹配（"谁能处理这种 URL"）→ 自建 Postgres 一张表**。`agent_workers` 表含 `agent_type`、`capabilities JSONB`、`heartbeat_at`、`endpoint`。50 行代码搞定注册+心跳+查询。比 Nacos 简单 10 倍，还自带事务能力。**绝大多数 P2 场景的最佳解**。

**需要跨系统 / 跨组织互操作 → AgentCard + Nacos/A2A 全套**。比如你要把 trip-os 的 `travel-analyzer` agent 暴露给别的系统调用、或者你要调用别人家的 agent 作为 subagent。这时候 AgentCard 的标准化 schema、Nacos 的版本锁定、A2A 的协议一致性才产生收益。

**从 Java 背景转入 → 反直觉警告：别一上来就上 Nacos**。Java 生态里 Nacos 是标配，但在 agent 场景里**k8s Service 已经是完整的"服务发现 + 健康检查 + 负载均衡"**，P1/P2 完全够用。只有当 agent 要对外暴露 / 跨系统互操作时（真正分布式系统属性），Nacos 这层才有价值。

**如果同时支持 Python runtime 与 Java/企业服务化 runtime → registry 抽象不能绑死语言栈**。AgentScope Python 2.x 给的是 session/workspace/message-bus/team runtime；AgentScope Java 给的是 AgentCard/A2A/Nacos/Spring 发布与发现。agent-os 应把 `AgentRegistry` 定义成语言无关契约，本地实现可以是内存或 Postgres，Java/企业实现再接 Nacos/A2A。

### 常见陷阱

- **把"注册中心"当万能，忘了能力声明才是独特价值**：没有 skills 字段、没有版本机制的 registry 等于 DNS。AgentCard 的真正价值不在"地址解析"而在"按能力查询"——"有没有哪个 agent 能做 OCR"这种问题，传统 DNS/k8s 答不了
- **心跳风暴**：Worker 数量多时所有心跳同时到 registry 会打爆后者。必须加 jitter 分散起始时间，或用被动心跳（连接存在即视为存活）
- **注册和反注册不对称**：进程 crash 不走正常退出流程 → 注册表残留僵尸 worker。**必须 TTL + 健康检查双保险**，不能只靠注销信号
- **Session 亲和性被忽略**：默认 round-robin 让 prompt cache 每次 miss，token 费用 ~10x。注册表必须支持 `(session_id → worker_id)` 的 sticky 路由，这是 agent 比传统服务多出来的维度
- **能力声明 vs prompt 指令混淆**：AgentCard 的 skills 字段声明的是 "worker **能做**什么"，不是 "主 agent **应该让它做**什么"。后者是编排决策，不该塞进 registry 里——这是两个关注点
- **版本锁定没做导致线上炸**：subagent agent 代码迭代时，主 agent 应该能选择"锁定调用 v1.2" vs "总是最新"。Nacos 原生支持版本锁定，自建 Postgres 表也要加 `version` 字段；否则灰度发布时主 agent 不可控地被推到新版
- **跨语言 / 跨框架互操作靠名字协商做不到**：真的要做跨系统 agent 互联，**必须走 AgentCard 这种结构化 schema + A2A 这种标准协议**。"我们都调 HTTP 就行" 是低估了消息格式、流式、身份、授权这些跨系统时的根本问题

## 和相邻概念的关系

- **[[multi-agent]]** —— 多 agent 协作的前置。没有 registry 就只能硬编码点对点布线，N² 连接问题无解
- **[[channel-remote]]** —— HTTP / WS 通道的前置。通道传什么协议（A2A / gRPC / 自定义 JSON）要先在 registry 里对齐
- **[[runtime-state]]** —— 注册信息本身是 runtime state 的一部分，持久化机制决定 registry 是内存 / SQL / Nacos
- **[[tool-system]]** —— tool registry 和 agent registry 是同一思想的两个层次；tool 是"agent 能调用什么"，agent 是"系统里有哪些 agent 可被调用"
- **[[mcp-skills]]** —— MCP 协议解决 tool 侧的标准化发现问题，和 A2A 在 agent 侧解决同样问题，两者哲学一致但层次不同

## L2 详情

- [[agent-registry-discovery--agentscope-java]] — **AgentCard + A2A + Nacos + Spring Boot** 的独立生产级参考
- [[multi-agent--agentscope]] — Python 2.x 的 session/workspace/message-bus/team runtime；历史 A2A/Nacos 内容不再代表当前 Python 源码
- [[multi-agent--openharness]] — `TeamRecord` 内存注册（反面参考：进程重启丢失的痛点）
- [[multi-agent--claude-code]] — `swarm/backends/registry` 是 agent runtime 后端注册，不是 agent 实例注册（术语相同但对象不同，需区分）
- [[multi-agent--deer-flow]] — `_background_tasks` 运行时任务表（单机）
- [[multi-agent--hermes-agent]] — `_active_children` 活跃子 agent 列表（单机）

## 参考来源

- Google A2A 协议：https://github.com/google/A2A
- Nacos 官方文档：https://nacos.io/en-us/docs/what-is-nacos.html
- AgentScope Java 源码：`/Users/neo/Desktop/project/git/agentscope-java`
- AgentScope Python 2.x 源码：`/Users/neo/Desktop/project/git/agentscope`（当前主线为 app/session/workspace/team runtime）
