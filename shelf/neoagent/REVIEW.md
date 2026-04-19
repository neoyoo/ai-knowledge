# Ingest Review：neoagent

**Ingest 时间**: 2026-04-19
**源路径**: `/Users/neo/Desktop/project/git/neoagent/`
**源版本**: `9d3621d3cf2717ee989dd954bd159d9d4c9a20db` (2026-04-13)
**kb_path**: `/Users/neo/Desktop/project/git/ai- knowledge/`
**状态**: `READY_TO_MERGE`

---

## 概览

| 指标 | 数值 |
|------|------|
| L2 文件数 | 10 |
| L1 patch 数 | 10 |
| 顶层模块覆盖率 | 100%（11/11） |
| 自动修复项 | 0 |
| 待确认问题 | 0 |
| 阻断问题 | 0 |

### 产出文件清单

**L2 页面**（全部 confidence=high）：

| 概念 | 文件 | confidence |
|------|------|-----------|
| query-loop | `drafts/neoagent/wiki/_impl/query-loop--neoagent.md` | high |
| prompt-system | `drafts/neoagent/wiki/_impl/prompt-system--neoagent.md` | high |
| tool-system | `drafts/neoagent/wiki/_impl/tool-system--neoagent.md` | high |
| context-management | `drafts/neoagent/wiki/_impl/context-management--neoagent.md` | high |
| memory-system | `drafts/neoagent/wiki/_impl/memory-system--neoagent.md` | high |
| session-recovery | `drafts/neoagent/wiki/_impl/session-recovery--neoagent.md` | high |
| hooks | `drafts/neoagent/wiki/_impl/hooks--neoagent.md` | high |
| channel-remote | `drafts/neoagent/wiki/_impl/channel-remote--neoagent.md` | high |
| multi-agent | `drafts/neoagent/wiki/_impl/multi-agent--neoagent.md` | high |
| evaluation-observability | `drafts/neoagent/wiki/_impl/evaluation-observability--neoagent.md` | high |

**L1 patches**（10 个，每个含 OP-1 + OP-2，部分含 OP-3）：

| 目标 L1 | OP-1（新增列） | OP-2（新场景/建议） | OP-3（跨概念关系） |
|---------|----------------|---------------------|-------------------|
| query-loop.md | ✓ | ✓ freed/recall + ContextVar 并发 | ✓ 增强 depends_on context-management 的 evidence |
| prompt-system.md | ✓ | ✓ 三态 skill + 循环级动态段 | 无新关系 |
| tool-system.md | ✓ | ✓ auto_free_after 协议 + learned/ 自扩展 | ✓ 新增 supports context-management |
| context-management.md | ✓ | ✓ 方案 E freed/recall + schema token budget | 无新关系（在 tool-system 的 OP-3 覆盖） |
| memory-system.md | ✓ | ✓ 可编辑 markdown + 项目级 hash | 无新关系 |
| session-recovery.md | ✓ | ✓ save_if_storage + ResumeWarningEvent + Session.fork | 无新关系 |
| hooks.md | ✓ | ✓ 双轨小规模 + hook 权限绕过防御 | 无新关系 |
| channel-remote.md | ✓ | ✓ asyncio.Queue 桥接 + Lock 串行 + stderr drain | 无新关系 |
| multi-agent.md | ✓ | ✓ 小而全框架 + WorkerEvent 嵌套 + spawn_worker | ✓ 新增 feeds evaluation-observability |
| evaluation-observability.md | ✓ | ✓ hooks+EventBus 正交 + 每 case 独立 collector + WorkerEvent 嵌套 | OP-3 在 multi-agent patch |

**分析文件**：

- `drafts/neoagent/analysis/module-map.md` — 模块到概念维度的映射表
- `drafts/neoagent/analysis/coverage-report.md` — 100% 覆盖率 + 概念交叉检查

---

## 自动修复清单

**无**——全部 L2 页面初稿即满足质量门禁：frontmatter 完整（title/category/parent/source/source_version/concept/confidence/created/updated）、含全部必含四章节（概述/架构分析/设计亮点/局限性/来源）、关键代码路径含真实函数签名 + 行号 + 文件路径、文件命名符合 `{concept}--{source}.md` 规范。

---

## 待确认问题清单

**无**——所有 L2 confidence=high，L1 patch 均包含必要的 OP-1 和 OP-2（禁止 TBD 占位符）。

---

## 阻断问题清单

**无**：
- 覆盖率 100% ≥ 80% PASS
- 无 L2 写入失败
- 无 L2 与 L1 结论相反
- 无 L2 缺必含章节

---

## 总结与后续建议

neoagent 是 Python async agent 框架里**设计密度最高**的源之一——6200 行 Python 实现了 10 个 ontology 概念全覆盖，其中三个设计点特别值得关注，建议 Neo 在后续 brainstorming 时参考：

1. **`BaseTool.auto_free_after` 协议字段驱动的 freed/recall 机制**：同类项目唯一将"何时忘记工具结果"做成工具协议一等公民的方案，形成 `free → recall → 一轮可见 → auto-free` 可重入循环。建议写成 `wiki/_insights/` 洞察卡。

2. **Session-scoped MCP promoted_tools + ContextVar session 传播**：FastAPI 多用户场景下的并发隔离最完整方案，建议在 `channel-remote` 和 `mcp-skills` 两个 L1 的 patches 强调。

3. **Hook + Event 的双轨正交分离**（decision + observation）：只用 400 行代码就实现了和 Claude Code（28+ event）、AgentScope（元类注入）等价的能力，是"小但对"哲学的标杆——值得加入 `hooks.md` 的方案对比表。

**合并建议**：当前全部 READY_TO_MERGE，可按 `drafts/` 目录结构直接合并——
1. `drafts/neoagent/wiki/_impl/*.md` → `wiki/_impl/`
2. 应用 10 个 L1 patches（推荐人工 review 每个 patch 的 OP-1 列插入位置 + OP-2 场景合并点是否和现有内容语义连贯）
3. 将 `drafts/neoagent/analysis/` 归档到 `wiki/_analysis/neoagent/` 或直接删除（本轮分析已定稿）
4. 移除 `drafts/neoagent/` 临时目录

由于 neoagent 的设计原创性高（尤其 freed/recall、session-scoped promotion、双轨正交 hook+event），建议合并到 `wiki/` 而非归档到 `shelf/`。
