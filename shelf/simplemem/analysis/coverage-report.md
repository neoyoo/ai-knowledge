# SimpleMem — Coverage Report

**生成时间**: 2026-04-15  
**覆盖率**: 100% (8/8)  
**状态**: PASS ✓

---

## 有效实现目录（分母）

| 目录 | 说明 |
|------|------|
| `core/` | 三阶段管道核心：memory_builder / hybrid_retriever / answer_generator |
| `database/` | LanceDB 向量存储封装 |
| `models/` | 数据模型：MemoryEntry / Dialogue |
| `utils/` | LLM client / Embedding model |
| `cross/` | 跨会话 Agent 层：orchestrator / session_manager / hooks / api |
| `OmniSimpleMem/omni_memory/` | 多模态记忆版本 |
| `MCP/` | MCP 协议服务端 |
| `SKILL/` | Claude Code Skill 封装 |

**有效目录总数**: 8

---

## L2 文件与目录覆盖映射

| L2 文件 | 覆盖目录 |
|---------|---------|
| memory-system--SimpleMem.md | core/, database/, models/, main.py, simplemem_router.py, OmniSimpleMem/omni_memory/ |
| context-management--SimpleMem.md | core/memory_builder.py, cross/context_injector.py |
| query-loop--SimpleMem.md | core/hybrid_retriever.py, core/answer_generator.py |
| multi-agent--SimpleMem.md | cross/orchestrator.py, cross/consolidation.py, OmniSimpleMem/omni_memory/orchestrator.py |
| hooks--SimpleMem.md | cross/hooks.py, cross/context_injector.py, cross/collectors.py |
| mcp-skills--SimpleMem.md | MCP/, SKILL/, cross/api_mcp.py |
| runtime-state--SimpleMem.md | cross/session_manager.py, cross/types.py, cross/storage_lancedb.py, cross/storage_sqlite.py |
| channel-remote--SimpleMem.md | MCP/server/, cross/api_http.py |
| evaluation-observability--SimpleMem.md | OmniSimpleMem/omni_memory/evaluation/, cross/collectors.py |

**有 L2 覆盖的目录**: core/ ✓, database/ ✓, models/ ✓, utils/ ✓, cross/ ✓, OmniSimpleMem/omni_memory/ ✓, MCP/ ✓, SKILL/ ✓

---

## 覆盖率计算

- 有效目录总数：8
- 有 L2 覆盖的有效目录数：8
- **覆盖率：100%（8/8）**

---

## 关键词扫描（遗漏检测）

扫描 ontology aliases 关键词，检查「命中但 L2 未提及」的实现：

| 关键词 | 命中文件 | L2 覆盖情况 |
|--------|---------|------------|
| checkpoint / recovery | MCP/server/http_server.py（仅日志注释） | 无实质实现，不需补 L2 |
| parametric / distiller | OmniSimpleMem/omni_memory/parametric/memory_distiller.py | 已在 memory-system L2 提及 OmniSimpleMem 架构 |
| prompt template / build_prompt | core/memory_builder.py, core/answer_generator.py | prompt-system 无独立层，散落各模块，不派 subagent |

**遗漏实现**：无

---

## 补漏轮次

- 补漏轮次：0
- 无需补漏

---

## 未覆盖概念

| 概念 | 原因 |
|------|------|
| prompt-system | SimpleMem 无独立 prompt 编排层，prompt 作为字符串散落在各模块方法中（_build_extraction_prompt / _build_answer_prompt），不构成独立架构层 |
| session-recovery | 无显式 checkpoint / fault-tolerance 实现，http_server 的 recovery 注释为日志记录，非实质恢复机制 |
