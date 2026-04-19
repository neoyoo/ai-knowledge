# trip-os KB 摩擦日志

> 真实使用 KB 过程中的摩擦记录。每条含：查询 → 去哪 → 命中度 → 缺什么。  
> 这份文件就是 kb-maintain 的真实 TODO 源。

---

## 2026-04-19 决策：Web 服务化后的沙箱 + bash + 下载安全

**决策文件**：[decisions/2026-04-19-web-service-security-sandbox.md](./decisions/2026-04-19-web-service-security-sandbox.md)

| 时间 | 查询 | 去哪 | 命中 | 摩擦 / 缺什么 |
|---|---|---|---|---|
| 12:47 | "多用户 Web 服务 + 沙箱总体架构" | `wiki/sandbox-isolation.md` | ✅ 好 | 威胁模型分档、gVisor/Firecracker 选型、硬化清单全齐。T3 段清晰 |
| 12:48 | "bash 执行安全——要做什么？" | `cookbook/tools/definitions/shell-execution.md` | 🟡 部分 | 对比了 3 家项目的 bash 设计，但**没有落地级别的"最小必须配置清单"**（应做：cap-drop / seccomp profile / non-root UID / 网络策略）。需要 cookbook 补"production bash sandbox checklist" |
| 12:50 | "文件下载的安全问题（SSRF、恶意内容）" | 没独立页 | ❌ 严重缺口 | 只有 `sandbox-isolation.md` 第 D 节第 1 条提了一句"元数据 IP 屏蔽"。**没有"download_media 安全"专题**：SSRF 拦截、scheme 白名单、size cap、MIME 校验、私网黑名单这些都没成页 |
| 12:52 | "agentscope-runtime 具体怎么接入 trip-os" | 没独立页 | ❌ 严重缺口 | `wiki/sandbox-isolation.md` 只说它是 gVisor 路径的实现代表。**上一轮 subagent 做的 800 字集成调研没有沉淀到 KB**——pip install 命令、BaseSandbox 最小示例、env 配置、P1/P2/P3 路径、未验证点——全在对话里 |
| 12:54 | "download_media 放沙箱外还是沙箱内？" | 没有 | ❌ 盲点 | 这是个**架构决策级别的 pattern**——run_python 断网、所有出站走 download_media 代理——我之前在对话里提过，但没成 pattern 页 |
| 12:55 | "run_python 沙箱的预热池怎么做？" | `wiki/sandbox-isolation.md` 第 D 节第 9 条 | 🟡 提了 | 有"预热池 + 心跳回收"一句话，但**没有怎么做的参考实现**。agentscope-runtime 的 `POOL_SIZE` / `HEARTBEAT_TIMEOUT` 机制没落地为 cookbook 条目 |

## 归类

| 类 | 条目 | 说明 |
|---|------|------|
| **A. 独立专题缺失** | Q3（download 安全）、Q4（agentscope-runtime 集成） | 需要新页 |
| **B. 落地指南缺失** | Q2（bash checklist）、Q6（预热池做法） | wiki 有方向，cookbook 缺模板 |
| **C. Pattern 候选** | Q5（egress-proxy-only network） | 架构级 pattern，还没写 |

## 生成的新 ideas

- [[../../ideas/2026-04-19-download-safety-as-separate-page]] — 文件下载安全作为独立 KB 页（Q3）
- [[../../ideas/2026-04-19-agentscope-runtime-cookbook]] — agentscope-runtime 集成作为 cookbook quickstart（Q4）
- [[../../ideas/2026-04-19-egress-proxy-only-pattern]] — "沙箱断网 + 代理层下载"作为 pattern（Q5）

（这三条待我补到 ideas/ 目录）

---

## 2026-04-19 决策：Web 版本的上下文注入与历史加载

| 时间 | 查询 | 去哪 | 命中 | 摩擦 / 缺什么 |
|---|---|---|---|---|
| 14:30 | "Web 版本 agent 的请求生命周期" | 没独立页 | ❌ 缺 | 需要跨 channel-remote + session-recovery + prompt-system + context-management 的**综合生命周期 pattern 页**（load → trim → inject → run → persist） |
| 14:32 | "中间件做历史加载" | `wiki/_impl/hooks--deer-flow.md` | 🟡 部分 | DeerFlow middleware 5 切入点讲得好，但**没专门讲"HTTP 请求内从 storage 加载 history 到 messages"这个具体 use case**。这是 web 场景的头号 middleware 却没明写 |
| 14:34 | "按 session key 缓存 agent 实例" | `wiki/channel-remote.md` 对比表 Hermes 列 | ✅ 找到 | Hermes `_agent_cache` 有完整讨论。prompt cache 经济学、config 签名失效都在 |
| 14:35 | "prompt cache 结构稳定怎么保证" | 没独立页，散落在多处 | 🟡 部分 | prompt-system 没有"prompt cache 友好的结构设计"专题。AgentScope 的 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 关键词提了但没展开 |
| 14:37 | "messages 增量持久化策略" | `wiki/session-recovery.md` OpenHarness 列 | 🟡 部分 | OpenHarness 提"每个 user turn 结束自动快照"。但没谈 **"SSE 流式边发边持久"** 的细粒度增量策略 |
| 14:39 | "存储分层（关系型/全文/对象）推荐" | 没有 | ❌ 盲点 | KB 完全没讨论 "Web 版本 agent 的存储架构"——Postgres/ES/MinIO 怎么分层 |

### 归类

- **A. 重要缺失 pattern**：**web-request-lifecycle-for-agent** 跨 4 概念，这是 trip-os 下一步最直接要的。值得写成 `wiki/_patterns/` 页
- **B. 颗粒度缺失**：prompt cache 友好设计、SSE 流式增量持久、存储分层——都是 Web 版本特有的具体决策，KB 没细讲
- **C. 已命中 (不算摩擦)**：`_agent_cache` 这条直接解决了关键费用陷阱

### 触发的新 idea

- `web-request-lifecycle-for-agent` pattern（跨 4 概念）→ 应加入 `ideas/sandbox-security.md` 或新开 `ideas/web-service-architecture.md`

---

## 2026-04-19 决策：分布式 multi-agent 派发（Web 版本 subagent 调度）

| 时间 | 查询 | 去哪 | 命中 | 摩擦 / 缺什么 |
|---|---|---|---|---|
| 15:10 | "分布式 subagent 任务派发" | `wiki/multi-agent.md` + `wiki/_impl/multi-agent--*.md`（全 6 家）| ❌ 最严重盲点 | **KB 里所有 multi-agent 参考都是单机的**。Claude Code 自己在对比表里写"Coordinator 目前是单机的，缺乏真正的分布式协调"。DeerFlow/Hermes 是 ThreadPoolExecutor，OpenHarness 是子进程——都单机 |
| 15:12 | "agent 主实例水平扩展 + 无状态部署" | 没有专题 | ❌ 零覆盖 | KB 完全没讨论"主 agent 无状态 + 负载均衡多 node"的架构模式 |
| 15:14 | "Session affinity / sticky session" | 没有 | ❌ 零覆盖 | session 路由到固定 node 的粘性设计零覆盖 |
| 15:15 | "跨进程/跨网络 agent 通信协议" | `wiki/_impl/multi-agent--agentscope.md`（A2AAgent）| 🟡 仅 1 家 | AgentScope 用 Google A2A 协议是唯一沾边的，但没展开跨机分布式的具体工程问题（服务发现、限流、超时） |
| 15:17 | "消息队列做任务派发（Celery/Redis Streams/Temporal）" | 没有 | ❌ 零覆盖 | 没有任何一家的参考提到 agent 任务走 message broker 的模式 |
| 15:19 | "Subagent 进度跨 node 回推给 client" | 没有专题 | ❌ 零覆盖 | Redis pub/sub + SSE relay 这类多 node SSE 架构零覆盖 |

### 归类

这是**从决策摩擦挤出来的最严重 KB 盲点**——不是"页面不够细"，是**整个分布式 agent 架构主题缺席**。

- **A. 重要缺失 pattern**：`distributed-agent-orchestration`（跨 multi-agent + channel-remote + runtime-state 至少 3 个概念）
- **B. 维度缺失**：`wiki/multi-agent.md` 的方案对比表 5 种方案全是单机的，**应该加第 6 种"Distributed Task Queue"**
- **C. L1 补档**：`wiki/channel-remote.md` 聚焦"外部接入渠道"，但**没讲 agent 内部多 node 通信/路由**——要么扩充 channel-remote，要么新增 L1 概念页

### 触发的新 idea

这是个大 idea，不是细粒度小点，应该开一个新的 `ideas/distributed-agent-architecture.md` domain（或合到 `web-service-architecture.md` 下），包含至少：
- 主/子 agent 分布式派发的 3 种模式（session affinity / task queue / independent services）
- 跨 node 状态共享（什么放 Postgres、什么放 Redis、什么放 MinIO）
- 跨 node SSE relay
- `wiki/multi-agent.md` 补第 6 种方案到对比表

---

## 2026-04-19 查询：A2A 是什么协议

| 时间 | 查询 | 去哪 | 命中 | 摩擦 |
|---|---|---|---|---|
| 15:45 | "agentscope A2A 协议详情" | `wiki/_impl/multi-agent--agentscope.md` | ✅ 好 | 架构、三种 resolver、局限全讲清楚了 |
| 15:47 | "A2A 底层 wire protocol（HTTP/gRPC？流式？）" | 未明写，我翻了 `raw/agentscope/src/agentscope/agent/_a2a_agent.py` 才确认 | 🟡 需补 | L2 提到"HTTP 发送"但没讲流式（`async for` + 双响应类型 A2AMessage/Task）。应在 L2 补"wire protocol"小节 |

---

## 2026-04-19 查询：Agent 注册与发现机制

| 时间 | 查询 | 去哪 | 命中 | 摩擦 |
|---|---|---|---|---|
| 16:05 | "Agent 注册机制 / service discovery" | `wiki/_impl/multi-agent--agentscope.md` + 5 个 `--*.md` 对比 | 🟡 部分 | **只有 AgentScope 讲了生产级 registry（AgentCard + Nacos）**。其他 5 家只有"运行时内存注册"（OpenHarness TeamRecord、DeerFlow _background_tasks 等），不是真 registry |
| 16:07 | "Agent registry 的决策维度（要不要？什么时候要？）" | 没有独立页 | ❌ 零覆盖 | **KB 没有"agent 注册中心 vs Java 服务发现"对比**。也没有"什么规模/场景需要 registry"的决策指南。对从 Java 转来的开发者价值极高，但空白 |
| 16:10 | "k8s Service 作为轻量 registry 的思路" | 没有 | ❌ 零覆盖 | 一般开发者第一反应就是 k8s Service/Ingress 做 LB 和发现，但 KB 没点出这是"最简单的 registry"——新人容易一上来就上 Nacos |

### 归类

- **A. 决策维度缺失**：`wiki/multi-agent.md` 的"方案对比"没把 **"是否需要 registry"** 纳入决策轴。应该加到场景决策指南里
- **B. 新概念候选**：考虑在 `wiki/` 加一篇 `agent-registry-discovery.md` 作为**独立 L1 概念**——它横跨 multi-agent、channel-remote、runtime-state，**且从 Neo 的 Java 视角看是熟悉的服务发现问题**。应该作为正式概念给它一席之地而不是埋在 multi-agent L2 里
- **C. 跨域类比缺失**：KB 里"agent 领域的 XXX 对应传统后端的 XXX"这类类比几乎没有。对从后端背景转入 agent 开发的人有很强引导价值

---

## 2026-04-19 查询：生产级分布式 agent 交互设计覆盖

| 时间 | 查询 | 去哪 | 命中 | 摩擦 |
|---|---|---|---|---|
| 16:45 | "生产级分布式 agent 交互设计完整内容" | wiki/ + cookbook/ 全域 | 🟡 30% 覆盖 | **只有 AgentScope 一家做到了注册发现 + A2A 跨进程 + 分布式 memory 后端**。其余 5 家明确承认单机 |
| 16:50 | "分布式任务队列派发（Celery/Temporal/Redis Streams）" | 没有 | ❌ 零 | 所有 6 家参考都是线程池/子进程，**没人做 broker-based 异步派发** |
| 16:52 | "跨 node 事件回传（SSE relay via Redis pub/sub）" | 没有 | ❌ 零 | 这是多 node Web 服务的关键能力，KB 零覆盖 |
| 16:54 | "agent 间路由策略（affinity / capability matching / LB）" | 没有 | ❌ 零 | AgentCard 里有 capability 字段，但**没有"根据 capability 做路由"的决策讨论** |
| 16:56 | "分布式追踪（跨 agent/跨 node 的 trace span）" | 🟡 部分 | 都是单进程 | Claude Code OpenTelemetry 和 AgentScope `@trace_*` 是**单进程** trace，没讲分布式 trace propagation |

### 归类

**KB 覆盖了"生产级分布式 agent 交互"的 30%**：
- ✅ 注册 + 发现（AgentCard + Nacos）
- ✅ 跨进程通信协议（A2A）
- 🟡 分布式状态（Redis/SQL 后端）
- ❌ 任务派发、事件回传、路由、熔断、分布式 trace 全缺

### 建议升级（ideas 里已经有 distributed-agent-architecture 候选）

1. **新 L1：`wiki/agent-registry-discovery.md`** — 抽 AgentScope 的 AgentCard/Nacos 作为独立概念，不再埋在 multi-agent L2 里。**信息密度够，可立写**
2. **新 pattern：`wiki/_patterns/distributed-agent-orchestration.md`** — 承载完全缺失的 50%（任务派发/事件回传/路由/熔断）。**但需要 trip-os 实跑后补参考实现，不能空写**
3. **`wiki/multi-agent.md` 补一列**：对比表加"分布式就绪度"这一维度，让读者一眼看出哪家能上多 node

---

## 2026-04-19 P0 基础设施：docker-compose 拉镜像失败

| 时间 | 现象 | 根因 | 解法 |
|---|---|---|---|
| - | `docker compose up -d` 拉 postgres:16-alpine 和 redis:7-alpine，Docker Hub 返回 `failed to copy: httpReadSeeker: failed open: failed to do request: ... EOF`，两个镜像都掉线 | Docker Desktop 默认不走系统代理（即不走 Clash 的 TUN 模式），直连 Docker Hub 在国内长连接容易被掐断 | Docker Desktop → Settings → Docker Engine，在 JSON 里加 `"registry-mirrors": ["https://docker.m.daocloud.io", "https://dockerproxy.com", "https://docker.nju.edu.cn", "https://hub-mirror.c.163.com"]`，Apply & Restart 后重新 `docker compose up -d` 秒拉完成 |

### KB 收获

1. **任何新起 Docker 环境**（不只是 trip-os），第一步都该先配 registry mirror
2. **`docker-compose.yaml` 不需要改任何配置**，mirror 是 daemon 层的事
3. **colima 配 mirror 的方式不同**（改 `~/.colima/default/colima.yaml` 的 `docker.registry-mirrors`），已经补到 `trip-os/infra/README.md`

→ 不升级（环境问题，非架构）
