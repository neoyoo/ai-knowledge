---
title: KB 治理相关想法
domain: kb-governance
updated: 2026-04-19
---

# KB 治理相关想法

关于知识库本身如何维护、自我进化、归档提升的想法。

---

## 2026-04-19 — shelf 机制增加"设计提升通道"

**status**: inbox  
**potential_target**: `schema/ingest-strategies.yaml` 规则更新 + 实际提升操作

**核心想法**：当前 `shelf/` 规则一刀切："项目不成熟 → 全部不进 wiki"。但**项目成熟度 ≠ 其中单点设计的创新度**。shelf 里可能埋着神器 pattern。需要一条通道：项目留 shelf，但其中独立设计能单独提到 `wiki/_insights/` 或 `wiki/_patterns/`。

**为什么值得**：neoagent 已验证——原 `auto_free_after` 协议（已于 2026-04-20 移除）被手工搬进 `wiki/_patterns/`，没这条通道就被埋没；simplemem 也做过类似提升（`wiki/_insights/simplemem--dual-storage.md`），说明需求真实只是没规范化。

**落地**：
1. `schema/ingest-strategies.yaml` 加"shelf 提升规则"——shelf 时必须在 `SHELF.md` 列"候选提升项"
2. 候选提升项自动进 `ideas/` inbox
3. `kb-maintain` 扫 SHELF.md 确保 ideas/ 有对应条目

**门槛**：同类源未见过 + 一句话能说清核心机制。

**相关**：`shelf/neoagent/SHELF.md`、`shelf/simplemem/SHELF.md`、`wiki/_insights/simplemem--dual-storage.md`

---

## 2026-04-19 — KB-first 查询实时记录摩擦

**status**: inbox  
**potential_target**: `kb-use-first` workflow skill + `projects/*/kb-friction.md` 规范

**核心想法**：KB 的真实 bug 只有用它才发现。每次决策/调研前先翻 KB，把"找不到 / 找错 / 不够深"实时记到 `projects/<proj>/kb-friction.md`。摩擦日志累积到一定程度自动成为 `kb-maintain` 的真实 TODO 源——不需要空想 lint 规则。

**为什么值得**：已经验证一次（2026-04-19 free/recall 用例 → 挤出 8 条摩擦归成 4 类真 bug，比空想 10 个 lint pass 具体得多）；传统 lint 找结构性 debt，摩擦日志找内容性 debt，两者互补；成本极低（反正要查 KB，顺手记一行）。

**落地**：
- **短期（手工）**：查询前 30 秒落一行（时间/查询/去哪/命中度/缺什么）
- **周期 review**：重复摩擦 = 高优 TODO
- **固化**：成为 `kb-use-first` skill 或 `kb-maintain` 的 pass

**风险**：纪律性要求高，不按习惯做就退化成装饰（参考 hermes——把"记录摩擦"变 prompt 硬规则）。

**相关**：hermes-agent `SKILLS_GUIDANCE`、本文件其它 idea、`projects/trip-os/kb-friction.md`（第一次实例）

---

## 2026-04-19 — 借鉴 hermes-agent 给 KB 装自进化机制

**status**: inbox  
**potential_target**: 三个新 skill（kb-maintain / kb-capture-practice / kb-background-review）的设计基础

**核心想法**：hermes-agent 的 "self-improving" 拆成 **2 轨 × 2 时机** 矩阵（即时 / 延迟 × memory / skill）。这个矩阵直接映射到 KB 维护——KB 条目 + skill 本身各有即时 patch 和延迟 review 两种形态。

**可直接搬的 5 个机制**：
1. **Nudge 计数器触发** — 每 N 次 KB 查询 / 决策 → trigger review。便宜、可预测
2. **Background Review Agent** — fork daemon thread 子 agent，共享存储但阈值设 0 防递归
3. **Skill patch 动作** — find-and-replace 小补丁，不必全量改写（注意加 uniqueness check，hermes 没做）
4. **GUIDANCE prompt 硬规则** — 把"何时沉淀"写进 system prompt（"don't wait to be asked"）
5. **写入前三道检查** — injection scan + credential leak + verdict 分级（我们可简化为一道凭证扫描）

**和 hermes 的差异**：hermes 自动写入，我们**坚持先进 `ideas/inbox` 人工审批**——KB 是长期资产，审一下值得。

**落地顺序**：
1. 先走 `kb-use-first` 工作流累积摩擦数据
2. 写 `kb-capture-practice` 最小版（手动触发 → ideas/inbox）
3. 写 `kb-background-review`（扫 ideas/、shelf、friction）
4. 最后 `kb-maintain`（nudge 触发 + patch 动作）

**关键代码参考**：hermes `run_agent.py:1718-1822`（`_spawn_background_review`）、`tools/skill_manager_tool.py:56-74`（security scan）、`agent/prompt_builder.py:164-171`（SKILLS_GUIDANCE）。

**KB 盲点**：现有 `wiki/_impl/memory-system--hermes-agent.md` + `mcp-skills--hermes-agent.md` 分别讲了 memory review 和 skill guidance，但没有一篇综合视角讲 hermes 的"self-evolution"——建议将来补 `wiki/_impl/self-evolution--hermes-agent.md`。

**相关**：hermes-agent 源码、`wiki/_impl/memory-system--hermes-agent.md`、本文件其它两条
