---
title: Web 服务化后的沙箱 + bash + 下载文件安全策略
date: 2026-04-19
status: proposed
decider: neo
related-wiki: [[sandbox-isolation]], [[channel-remote]], [[tool-system]]
related-patterns: [[_patterns/tool-metadata-driven-context-lifecycle]]
---

## 背景

trip-os 要从命令行本地工具升级为对外 Web 服务，外部用户通过 HTTP 把图片/链接送进来让 agent 分析。威胁模型从 **T1 本机单用户** 跳到 **T3 多租户 SaaS**（分级标准见 [[sandbox-isolation#威胁模型分层]]）。

三条具体担忧：
1. 多用户隔离——A 用户的代码能不能影响/读到 B 用户？
2. `run_python` / bash 的执行安全——LLM 被恶意页面注入怎么办
3. download_media 的文件下载安全——SSRF、恶意大文件、metadata IP

## 候选方案

### 沙箱底座（从 [[sandbox-isolation]] 直接采）

| 方案 | 威胁模型匹配 | 启动延迟 | 运维复杂度 | 成本 |
|------|------------|---------|----------|------|
| A. 纯 subprocess + env 白名单（现状） | T1 | ~instant | 零 | 零 | ❌ T3 不够 |
| B. Docker + cap-drop + seccomp | T2 | 秒级 | 低 | 低 | ❌ 共享宿主 kernel，T3 不够 |
| **C. Docker + gVisor（runsc）** | T3 | ~150ms（预热池） | 中 | 中 | ✅ 务实选择 |
| D. Firecracker microVM | T3 极致 | ~150ms | 高（需要 Nomad/集群） | 中高 | ⚠️ 我们不需要这么硬 |
| E. E2B SaaS | T3 | ~150ms | 零（外包） | 按秒付费 | ⚠️ 依赖第三方，数据过他们手 |

### 运行时实现（在 C 方案下）

| 方案 | 说明 | 推荐 |
|------|-----|------|
| **agentscope-runtime 1.1.4** | 一行 `CONTAINER_DEPLOYMENT=gvisor` 切换后端；自带预热池 + 心跳回收 + 持久 IPython kernel；Apache 2.0 | ✅ |
| 自建 `docker run --runtime=runsc` 包装 | 所有预热池/生命周期/MCP 暴露都自己写 | ❌ 重复造轮子 |

## 决策

**采用方案 C + agentscope-runtime**，分阶段上：

- **Phase 1**（P1，可立即做）：本地 Docker 后端，先把现有 subprocess 弱沙箱换成 agentscope-runtime 的 BaseSandbox。`run_python` 工具签名不变，用户无感
- **Phase 2**（P2，Web 服务上线前）：`CONTAINER_DEPLOYMENT=gvisor`，宿主装 runsc；启用 session-scoped 容器（per-session 隔离）
- **Phase 3**（P3，规模化时）：迁阿里云 ACK，`CONTAINER_DEPLOYMENT=k8s`

## download_media 的处理（架构决策）

**把 download_media 移出沙箱，独立代理服务**：

- `run_python` 沙箱 **egress 默认断网**
- 需要抓外部数据时 LLM 只能调 `download_media(url)`
- 代理服务在沙箱外做下列检查，然后把文件塞进 sandbox 挂载的 workspace
- 沙箱内 Python 只能处理这些已落地的数据，无法主动发起外部请求

**download_media 的检查清单**（SSRF 防护基础）：
1. Scheme 白名单：只允许 `http` / `https`
2. IP 黑名单：解析域名后拒绝私网地址（10.*、172.16-31.*、192.168.*、169.254.*、127.*、::1、fc00::/7）
3. **云元数据 IP 专项屏蔽**：169.254.169.254、fd00:ec2::254、metadata.google.internal（SSRF 第一大向量）
4. MIME 白名单：image/*、application/pdf 等；HTML/SVG 强制 Content-Disposition: attachment（参考 DeerFlow 做法）
5. Size cap：50MB 上限，超出就 cut
6. Timeout：30s

**为什么这样设计（核心直觉）**：把"发起网络请求的能力"从 LLM 手里拿走、移到我们能严格审计的代理层。沙箱内 Python 能力降为"处理已提供的数据"，外泄面大幅收窄。

## 依据

- [[sandbox-isolation]] 威胁模型分层：T3 必须内核级隔离，Docker 单独不够
- [[sandbox-isolation]] gVisor 专题：性能足够（syscall 密集型慢 2-5x，CPU-bound 几乎无感）、OCI 无缝
- 2026-04-19 subagent 调研报告：agentscope-runtime 1.1.4 已 PyPI 发布、支持 `CONTAINER_DEPLOYMENT=docker/gvisor/k8s` 环境变量切换、最小集成两行代码（`BaseSandbox()` context manager）
- `wiki/_patterns/tool-metadata-driven-context-lifecycle` 协议保留：run_python 仍有 `auto_free_after`，和沙箱底座换皮无关

## 未知风险 / 待验证

1. agentscope-runtime Docker 后端**有没有 egress 默认关的配置**——文档没找到；要么自己用 Docker network 做，要么验证后写到我们的 wiki
2. Docker 后端下 per-container **CPU/内存限制**——文档只在云端后端段落提到，本地 Docker 是否支持不明
3. `run_ipython_cell` **per-call 超时**文档未说，长任务行为待实测
4. 预热池 `POOL_SIZE` 在 macOS 开发机上的实际内存占用
5. 迁阿里云 ACK 时，容器镜像仓库需要自建还是可以用阿里云 ACR 托管
6. SSE 流式 + 沙箱的组合——agent 进度 event 怎么跨沙箱边界回到主 agent

## 下一步

- P1.0（30 分钟）：本地 colima + docker + pull agentscope runtime base image，跑通 BaseSandbox 最小 demo
- P1.1（2 小时）：实测上面未知风险 1-3，写到 `projects/trip-os/open-questions/`
- P1.2（2 小时）：改 `RunPythonTool`，把 subprocess 替换为 agentscope-runtime（参考 [[_patterns/tool-metadata-driven-context-lifecycle]]，`auto_free_after` 协议保留）
- **KB 补全**：把这次决策里暴露的 3 条 friction（download 安全专题、agentscope-runtime cookbook、egress-proxy-only pattern）按 ideas 流程走
