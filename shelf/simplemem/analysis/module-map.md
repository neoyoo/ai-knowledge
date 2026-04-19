# SimpleMem — Module Map

**生成时间**: 2026-04-15  
**源路径**: /Users/neo/Desktop/project/git/SimpleMem  
**主语言**: Python  
**代码规模**: ~160 个 .py 文件，约 45,963 行代码

---

## 顶层目录扫描（排除 .git、__pycache__、node_modules）

| 目录/文件 | 类型 | 描述 |
|-----------|------|------|
| `core/` | 实现 | 三阶段记忆管道核心：memory_builder、hybrid_retriever、answer_generator |
| `database/` | 实现 | 向量存储（LanceDB 封装）|
| `models/` | 实现 | 数据模型：MemoryEntry、Dialogue |
| `utils/` | 实现 | LLM client、Embedding model 工具 |
| `cross/` | 实现 | 跨会话 Agent 层：orchestrator、session_manager、hooks、api_http/mcp、storage |
| `OmniSimpleMem/` | 实现 | 多模态版本：omni_memory（text/image/audio/video）|
| `MCP/` | 实现 | MCP 协议服务端 |
| `SKILL/` | 实现 | Claude Code skill 封装 |
| `main.py` | 实现 | SimpleMemSystem 主类入口 |
| `simplemem_router.py` | 实现 | 统一路由器，Registry 模式 |
| `config.py.example` | 配置 | 配置模板 |
| `docs/` | 文档 | 说明文档 |
| `tests/` | 测试 | 测试套件 |
| `scripts/` | 脚本 | 工具脚本 |
| `docker-compose.yml` / `Dockerfile` | 配置 | 容器化部署 |
| `requirements*.txt` | 配置 | 依赖 |
| `README.md` | 文档 | 项目说明 |

---

## 有效实现目录（用于覆盖率计算）

| 目录 | 是否有效 |
|------|---------|
| `core/` | ✓ |
| `database/` | ✓ |
| `models/` | ✓ |
| `utils/` | ✓ |
| `cross/` | ✓ |
| `OmniSimpleMem/omni_memory/` | ✓ |
| `MCP/` | ✓ |
| `SKILL/` | ✓ |

排除目录: `docs/`, `tests/`, `scripts/`, `.github/`, `fig/`, `test_ref/`

---

## 概念维度覆盖计划

| 概念维度 (id) | 相关模块 | 是否派 subagent | 优先级 |
|--------------|---------|---------------|-------|
| memory-system | core/, database/, models/, main.py, simplemem_router.py, OmniSimpleMem/omni_memory/ | 是 | P0 |
| context-management | core/memory_builder.py, core/hybrid_retriever.py | 是 | P0 |
| query-loop | core/hybrid_retriever.py, core/answer_generator.py | 是 | P1 |
| multi-agent | cross/orchestrator.py, cross/consolidation.py, OmniSimpleMem/omni_memory/orchestrator.py | 是 | P1 |
| hooks | cross/hooks.py, cross/context_injector.py | 是 | P1 |
| mcp-skills | MCP/, SKILL/, cross/api_mcp.py | 是 | P1 |
| evaluation-observability | OmniSimpleMem/omni_memory/evaluation/, cross/collectors.py, tests/ | 是 | P2 |
| runtime-state | cross/session_manager.py, cross/types.py, cross/storage_lancedb.py, cross/storage_sqlite.py | 是 | P1 |
| channel-remote | MCP/server/, cross/api_http.py, cross/api_mcp.py | 是 | P2 |

未覆盖概念（无对应实现）:
- `prompt-system`: SimpleMem 无独立 prompt 编排层（prompts 散落在各模块中）→ 不派 subagent
- `session-recovery`: 无显式 checkpoint/fault-tolerance 机制 → 不派 subagent

---

## 排除模块列表

- `docs/` — 纯文档
- `tests/` — 测试（覆盖率计算时排除，但 subagent 可参考）
- `scripts/` — 工具脚本
- `fig/` — 图片资源
- `test_ref/` — 测试参考数据
- `docker-compose.yml`, `Dockerfile` — 部署配置
- `requirements*.txt` — 依赖配置
