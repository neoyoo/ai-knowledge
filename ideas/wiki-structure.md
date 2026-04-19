---
title: wiki 本身结构调整想法
domain: wiki-structure
updated: 2026-04-19
---

# wiki 本身结构调整想法

关于 wiki 主干页面本身的结构性改进（维度缺失、颗粒度不足、对比表补档等）。

---

## 2026-04-19 — `wiki/context-management.md` 对比表加"折叠-可召回"档

**status**: inbox  
**potential_target**: `wiki/context-management.md` 本身的修改

**核心想法**：`wiki/context-management.md` 对比表目前按 5 家项目列出方案：压缩 / 滑窗截断 / 摘要 / 蒸馏 / 三层截断。**缺少第四种哲学**——**"折叠-可召回"**：不压缩（保真）、不删除（可恢复）、粒度到单个工具调用。这是 neoagent `auto_free_after` 代表的思路，已在 `wiki/_patterns/tool-metadata-driven-context-lifecycle.md` 作为组合模式收录。但 L1 对比表**没体现这个方案存在**，从 L1 查找的人看不到这条路径。

**为什么值得**：**KB 第一次使用（2026-04-19 free/recall 用例）就暴露了这问题**——差点错过；是**方案空间漏洞**（补一个维度），不是"补一家项目"。

**落地**：改 `wiki/context-management.md`：
1. 对比表新增列/行"折叠-可召回"
2. 设计权衡章节加一行方案对比："折叠-可召回 | 工具协议自声明老化 | tool_result 大但偶尔 recall | neoagent"
3. 补一段讨论"何时选折叠 vs 压缩/截断"（粒度 per-tool vs per-message、保真度、召回成本）
4. 末尾"相关"加 wikilink 到 `wiki/_patterns/tool-metadata-driven-context-lifecycle`

**风险**：neoagent 被 shelf 了，L1 对比表直接列 "neoagent" 作代表源可能和"shelf 不进 wiki"规则冲突。**解法**：对比表引用 `_patterns/tool-metadata-driven-context-lifecycle` 作为代表源，pattern 可 reference shelf 项目，pattern 自己住在 wiki 主干——这也验证了"shelf 神器通过 pattern 提升"的通道（见 `kb-governance.md` 的 shelf-promotion-channel）。

**相关**：`wiki/context-management.md`（要改）、`wiki/_patterns/tool-metadata-driven-context-lifecycle.md`、`ideas/kb-governance.md#shelf-promotion-channel`
