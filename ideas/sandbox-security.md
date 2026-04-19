---
title: 沙箱与安全相关想法
domain: sandbox-security
updated: 2026-04-19
---

# 沙箱与安全相关想法

从 2026-04-19 trip-os 安全沙箱决策的 KB-first 摩擦中产生。核心目标：让 KB 对"Web 服务化 agent 的沙箱选型和代码/下载安全"有完整指南。

---

## 2026-04-19 — 文件下载安全作为独立 KB 页

**status**: inbox  
**potential_target**: `cookbook/tools/patterns/download-safety-checklist.md` 或 `wiki/_patterns/egress-proxy-download.md`

**核心想法**：当前 KB 只在 `wiki/sandbox-isolation.md` 第 D 节第 1 条有一句"屏蔽元数据 IP"。**"agent 下载文件"的安全问题比"agent 执行代码"更容易被忽略，但风险同级**——SSRF 撞元数据服务一条就能拿云端凭证。应该有一篇专页。

**为什么值得**：2026-04-19 trip-os 决策时 KB 找不到，全凭经验写；这套清单是**可迁移的 checklist**，所有 agent 下载工具都该过一遍；多场景适用（trip-os download_media、scrape agent、RAG ingest、浏览器 agent）。

**内容骨架**：
- 威胁清单（SSRF / 大文件 DoS / MIME sniffing XSS / 重定向到 file:// / DNS rebinding）
- 防护 checklist（scheme 白名单 / 私网 IP 黑名单 / 元数据 IP 专项 / MIME 白 / size cap / 301 重定向次数 / 文件名 sanitize）
- 跨项目对比（DeerFlow 强制 attachment / AutoGen 无 / Claude Code WebFetch 基本检查）
- 参考实现代码片段
- 陷阱（DNS rebinding / 只检查输入 URL 不检查最终 URL）

**落地**：写成 `cookbook/tools/patterns/download-safety-checklist.md`，在 `wiki/sandbox-isolation.md` link 过去。

**相关**：`wiki/sandbox-isolation.md`、`projects/trip-os/decisions/2026-04-19-web-service-security-sandbox.md`、本文件其它两条

---

## 2026-04-19 — agentscope-runtime 集成作为 cookbook quickstart

**status**: inbox  
**potential_target**: `cookbook/tools/quickstarts/agentscope-runtime-sandbox.md`

**核心想法**：前一轮 subagent 做的 800 字 agentscope-runtime 集成调研（pip install、`BaseSandbox` 示例、env 配置、P1/P2/P3 路径、4 条未验证点）**只存在于对话历史**。下次谁接入得重新调研——KB 失职。应沉淀成 cookbook quickstart。

**为什么值得**：2026-04-19 trip-os 决策时 KB 查不到落地信息；agentscope-runtime 1.1.4 刚发（2026-04-16）官方文档还没补齐，"我们自己踩过的第一手指南"价值高；与 `wiki/sandbox-isolation`（原理层）互补。

**内容骨架**：宿主机准备（colima + Rosetta）/ 镜像拉取 / 最小示例（BaseSandbox + run_ipython_cell）/ SandboxService 多 session / 后端切换（docker/gvisor/k8s）/ 集成进 neoagent（方案 A 替换 subprocess、方案 B 注册 MCP）/ 生产路径（POOL_SIZE / HEARTBEAT_TIMEOUT / WORKERS）/ 未验证点清单。

**落地原则**：**不实测不写**——等 trip-os 真跑通 P1 再沉淀，抄文档的指南价值低。

**相关**：2026-04-19 subagent 调研报告（对话历史）、`wiki/sandbox-isolation.md`、`projects/trip-os/decisions/2026-04-19-web-service-security-sandbox.md`、本文件其它两条

---

## 2026-04-19 — "egress-proxy-only network" 作为跨概念 pattern

**status**: inbox（等参考实现，incubating 候选）  
**potential_target**: `wiki/_patterns/egress-proxy-only-network.md`

**核心想法**：沙箱内 `run_python` **egress 默认完全断网**，所有外部访问必须走沙箱外的 `download_media` 代理。这是跨 L1 概念的组合模式（sandbox-isolation + tool-system），关键直觉：把"主动发起网络请求的能力"从 LLM 手里拿走。

**为什么值得**：一次解决 T3 多租户两大问题（SSRF + 数据外泄）；代理层集中做安全检查，逻辑不散落；对 LLM 是"温和降级"（仍能获取数据但通过受控接口）；可迁移（RAG ingest、浏览器 agent、scraper 都适用）。

**为什么是 pattern 而不是单页修改**：跨 2 个 L1 概念 ✅ / 可迁移 ✅ / 即将有参考实现 🟡 / 解决具体问题 ✅ —— 满足 `wiki/_patterns/_index.md` 收录标准。

**内容骨架**（pattern 页）：触发问题 / 参与的概念（sandbox-isolation + tool-system）/ 协议（6 步：egress deny → 代理工具 → 沙箱外做 SSRF+size+MIME → 落盘到挂载 workspace → 沙箱内只处理已落地数据）/ 何时不该用（T1 本地、需要低延迟、非 HTTP 场景）/ 参考实现。

**落地原则**：**没有参考实现的 pattern 只能是空话**。等 trip-os 真跑起来再写。验证前置：agentscope-runtime Docker 后端到底能不能默认 deny egress。

**相关**：`projects/trip-os/decisions/2026-04-19-web-service-security-sandbox.md`、`wiki/sandbox-isolation.md`、`wiki/tool-system.md`、本文件其它两条
