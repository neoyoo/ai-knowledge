# 决策：tool-result 过大用 cap + nudge，不做 stash + recall

- 日期：2026-05-28
- 项目：agent-os
- 状态：已定（落入 spec `docs/superpowers/specs/2026-05-28-tool-result-token-budget-design.md`）

## 背景

agent-os 生产硬化第一块要解决"单个工具结果过大撑爆 context"。初版设计走 neoagent 式 **stash + recall**：摄入时截断 head+tail，完整内容存 `ToolResultOverflowStore`，新增 `recall_tool_result` 工具按需召回，随 snapshot 持久化（bump version）。工作量 1-2 天。

## 决策

否决 stash + recall，改用 **Claude Code 式 cap + nudge**：每个工具结果设 token cap（默认 25000，env/per-tool 可覆盖），溢出时返回**极小 nudge tool-result**（~150 字节，引导模型缩小范围/分页重读），完整与截断内容都不进 context。无 store、无 recall 工具、不动持久化层。工作量降到约半天。

## 依据（Claude Code 源码 + 线上 A/B 实证）

源码：`/Users/neo/Desktop/project/git/claude-code-sourcemap/restored-src/src/`

- `tools/FileReadTool/limits.ts:1-14` —— 原注释记录线上 A/B：**试过"截断代替抛错"，工具错误率降但 mean token 升**（抛错 ~100 字节，截断 ~25K token 灌 context），**已回退**保留抛错。这是反对"截断塞回 context"的直接实证。
- `tools/FileReadTool/FileReadTool.ts:175-185` —— `MaxFileReadTokenExceededError` nudge 文本：引导 offset/limit 或 search。
- `tools/GrepTool/GrepTool.ts:80-81,104-119` —— `head_limit` 默认 250，"large result sets waste context"，`head_limit=0` 才解除。证明 cap+nudge 是全工具系统一致设计，非 Read 特例。

## 判据（什么时候反过来要 recall）

cap + nudge 成立的前提是工具**幂等、可廉价重读**（读文件、grep、查询）——agent 工具绝大多数如此，源还在，重读即可。
stash + recall 只在工具**非幂等 / 昂贵 / 一次性**（随机、流式、计费 API、副作用）时才有价值。届时作为 per-tool 可选增量再加，不是默认。

## 关联

- KB 摩擦：`projects/agent-os/kb-friction.md` 2026-05-28 条目（tool-system--claude-code 缺 Read 输出限制机制 + 缺"source re-read vs stash+recall"哲学对比）。本决策的源码事实可回流补 `wiki/_impl/tool-system--claude-code.md` 或新 `wiki/_insights/`。
- 对立 pattern：`wiki/_patterns/tool-metadata-driven-context-lifecycle.md`（neoagent stash+recall 四层）。两者不是对错，是按"工具是否幂等"分场景。
