# KB 升级计划 · 2026-05-28

> 触发：原调研的开源源项目均已迭代，本地 clone 停在 4 月初/中，上游已跑远。本计划基于 fetch 后的 commit delta + 各源 L2 现状对照得出。
> 状态：**计划阶段**，未动手。claude-code 本轮跳过（sourcemap 反编译源，需重新 restore，独立工程）。

---

## 1. 落差全景

| 源 | ingest 基线 → 上游现状 | 落后 commit | 价值 | 工作量 | 批次 |
|---|---|---|---|---|---|
| agentscope | 2026-04-12 → **v2.0.0** | 46 | 高（填 sandbox 空白 + 2.0 大版本） | 中 | **B1** |
| openharness | 0.1.0 → ~0.1.10-dev | 385 | 高（memory 从「零持久」变结构化 runtime） | 中 | **B2** |
| hermes | **v0.8.0 → v0.15.0** | 6310 | 最高（架构重构 + 多线新增） | 大 | **B3（单独立项）** |
| mempalace | 3.x → **3.3.6** | 1043 | 中高（图谱从静态变 living-memory） | 大（单页 40%+） | **B4** |
| deer-flow | 2.0 → 新 | 344 | 中（持久化层 + prefix-cache 架构） | 中 | **B5** |
| simplemem | → **0.3.0** | 30 | 低（核心引擎未变，新增独立子项目） | 小/可选 | **B6（可选）** |
| claude-code | 2.1.88 | — | — | — | 本轮跳过 |

**推荐执行顺序**：B1 → B2 →（B3 单独一轮）→ B5 → B4 → B6。
理由：B1/B2 量适中、各有清晰的架构级新增，先拿下；B3 体量最大须独占一轮；B4 虽单页但改动深、且涉及 benchmark 数据重测；B5 噪音多但有干货；B6 几乎可跳过。

---

## 2. SOP 红线（每批都必须守）

1. **禁止只改 L2 不更 L1**：每个受影响概念，L2 改完必须回写对应 L1 的「各家设计对比表」+「设计权衡」。这是价值所在。
2. **frontmatter 基线更新**：每个 L2 页头部 `source_version` / `updated` 必须刷成本次上游版本（见 §4 基线表）。
3. **覆盖率扫描**：每批 ingest 后跑一次目录扫描 + 概念交叉检查，防注意力遗漏。
4. **派 subagent 防爆**：单 agent output ≤15k token。大页（query-loop--hermes、sandbox--agentscope、memory--openharness、memory--mempalace）由主线程先写骨架，subagent 只填实现段；hermes 整批按页拆多个 subagent 顺序跑，每页独立 commit。
5. 完成后更新 `docs/sdk-kb-alignment.md`（新源版本 + 对 neoagent 下一版的设计影响）。

---

## 3. 分批详细计划

### B1 · agentscope → v2.0.0（中）

**受影响页（改）**
- `mcp-skills--agentscope.md` —【中·重写类层次】三类客户端（`HttpStatelessClient`/`StdIOStatefulClient`/`HttpStatefulClient`）已合并为单一 `MCPClient`（`mcp/_mcp_client.py`），现有「类层次」整章过期。
- `evaluation-observability--agentscope.md` —【小】tracing 从 `tracing/_setup.py` 全局装饰器迁到 `middleware/_tracing/`，须实例化 `TracingMiddleware` 传给 Agent；补 `Msg.usage`（token 追踪不依赖 OTel 的第二条路径）。
- `hooks--agentscope.md` —【小】元类 pre/post hook 描述仍对，但漏了新的 **middleware 五点**（`on_reply/on_reasoning/on_acting/on_model_call/on_system_prompt`），两套扩展机制并存须说清。
- `tool-system--agentscope.md` —【小】补 `ToolGroup` 独立类 + **Permission Engine**（工具执行前路径/危险命令检查，`.env` 保护，dangerous-path API）。
- `context-management--agentscope.md` —【小】补 acting 后的 `compact_tool_result`（独立于 `compress_context` 的大 tool result 压缩）。
- `channel-remote--agentscope.md` —【中】新增「FastAPI Agent Service」段（`app/`：WorkspaceManager / SessionManager / AgentMiddlewareFactory / AgentToolFactory 扩展点）。

**新建页**
- ⭐ `wiki/_impl/sandbox-isolation--agentscope.md` —【大】**填补 `sandbox-isolation` L1 至今零 L2 的空白**。`workspace/`：`WorkspaceBase` → `LocalWorkspace` / `DockerWorkspace` / `E2BWorkspace`，统一管 MCPs/Skills/Tools/Offload，并已集成进 `Agent.__init__`（Offloader）。

**L1 回写**：sandbox-isolation（首次进对比表）、mcp-skills、hooks、tool-system、context-management、evaluation-observability、channel-remote 的对比表各加/改 agentscope 列。

**执行**：主线程写 `sandbox-isolation--agentscope.md` 骨架 + 对比表结构 → 1 个 subagent 填 workspace 实现段；其余 5 页小改合并 1~2 个 subagent 顺序处理。

---

### B2 · openharness → ~0.1.10-dev（中）

**受影响页（改）**
- ⭐ `memory-system--openharness.md` —【大·重写】原定位「零持久化、仅元数据词法匹配、无后台提取」**已根本性过期**：
  - #266 schema-v1 frontmatter（稳定 id/签名/TTL/soft-delete/类型/scope）+ `usage_index.json` 使用频次；搜索改为元数据 2x / 正文 1x / 频次 0.1x 多维评分。
  - #267 Claude-style **agent-scoped memory runtime**：三级 scope（user/project/local）、快照初始化、session-memory 服务、`extract_memories_from_turn()` 自动提取、team vault + secret guard。
  - #246 **auto-dream consolidation**（后台自动蒸馏会话记忆）。
  - 过期具体行：原 L14（仅元数据匹配）、L42-45（约 20 行核心逻辑设计亮点）、L53（无后台自动提取）。
- `hooks--openharness.md` —【小】hook schema 4 类均新增 `priority: int`，executor 按优先级降序执行。
- `multi-agent--openharness.md` —【小】补 swarm 子进程继承 permission_mode + `OPENHARNESS_*` scoped env 转发（`swarm/spawn_utils.py`）。
- `runtime-state--openharness.md` / `query-loop--openharness.md` —【小】`close_runtime()` shutdown 关闭各 API client。

**新建页（候选，建议先折叠）**
- `memory-schema--openharness.md`：schema-v1 格式素材已够独立成页。**建议先折叠进 memory-system L2 的一节**；若 §5 决定立 memory-schema 为跨源新维度再抽出。

**L1 回写**：memory-system（openharness 列从「无持久」改为「结构化 runtime + auto-dream」——这会显著改变 memory-system L1 对比表的格局）、hooks、multi-agent。

**执行**：主线程写 memory-system--openharness 新骨架 → 1 subagent 填 agent/team/schema/usage/auto-dream 实现；其余小改 1 subagent。

---

### B3 · hermes → v0.15.0（大 · 单独一轮）

> 6310 commit、跨 7 个 minor。靠版本叙事而非全量 diff。主线：`run_agent.py` 2400 行单体 → `agent/` 子包；memory/skills/security/kanban 四线新增。**整批按页拆 subagent 顺序执行，每页独立 commit。**

**改动最重三页**
- ⭐ `query-loop--hermes-agent.md` —【高·重写架构叙事】核心论断「2400 行高密度单函数」**已失效**。现拆为 `conversation_loop.py` / `tool_executor.py` / `system_prompt.py` / `conversation_compression.py` / `stream_diag.py` / `agent_init.py` / `background_review.py` 等。L229-233、L257 行号引用全部失效。
- `memory-system--hermes-agent.md` —【中高】Hindsight `recall_types` 默认收窄为 observation only；`flush_memories` 删除；`on_session_reset` 删除；新增 provider 拿到 completed-turn 完整消息上下文。四处均改。
- `multi-agent--hermes-agent.md` —【中高】kanban 进化为完整多 agent 调度平台：dispatcher 派 review agents、stale 检测、scheduled 状态、per-task `model_override`、swarm 拓扑 helper、ACP session_id 烙印。当前页几乎无 kanban 内容，须大段新增。

**手术式修订页**
- `tool-system--hermes-agent.md`：`tools/web_providers/`、`tools/browser_providers/` 目录已删（插件化迁移）；新增运行时安全层——promptware 防御（共享威胁模式 + 内存加载扫描 + tool-result 定界符）、OSV.dev 供应链审计、secret redaction 默认开、插件工具可覆盖内置工具。
- `mcp-skills--hermes-agent.md`：skills bundles（`/<name>` 一次加载多 skill）；BrowseSh / HuggingFace tap；skills-hub 健康检查 + freshness badge + watchdog cron；目录 858→19932；REST `GET /v1/skills`、`/v1/toolsets`。
- `channel-remote--hermes-agent.md`：Dashboard 完整 OAuth（WS ticket / Nous plugin / audit log）；Docker s6-overlay 替 tini 作 PID 1 + per-profile supervision；外部 context engine 插件槽。
- `session-recovery--hermes-agent.md`：`_save_session_log` / `_clean_session_content` / `session_log_file` 全删；`session_search` 重构为单形状（discovery/scroll/browse）；`/exit --delete`；opt-in per-session JSON snapshot。
- `prompt-system--hermes-agent.md` / `context-management--hermes-agent.md`：跨 session 1 小时 prefix cache（Claude/OpenRouter/Nous）；`protect_first_n` 可配；context-engine host contract。
- `hooks--hermes-agent.md`：PluginContext 新增 `register_dashboard_auth_provider` / `register_transcription_provider`(STT) / `register_tts_provider` / `register_auxiliary_task()`。
- `runtime-state--hermes-agent.md`：`/model` 与 `hermes model` 统一 + 磁盘缓存；`/yolo` 正式进 session bypass 语义 + 视觉指示器。

**L1 回写**：query-loop（单体→模块化的叙事变化值得进设计权衡）、memory-system、multi-agent、tool-system、mcp-skills、channel-remote、session-recovery、prompt-system、context-management、hooks、runtime-state —— 几乎全套。

---

### B4 · mempalace → 3.3.6（大 · 单页深改）

> 仅 `memory-system--mempalace.md` 一页，但改动占全页 40%+，且图谱模型本质变化。

**过期论断**
- 「palace_graph 从 ChromaDB metadata 动态构建、无独立图存储」→ 现 hallways/tunnels 有独立权重存储层（`dynamics.py`）。
- 「`_TUNNEL_FILE` 硬编码 `~/.mempalace/tunnels.json`」→ 改为 `_get_tunnel_file(config)` 尊重 `palace_path`。
- 「embedding 用 `all-MiniLM-L6-v2`」→ 新装默认 `embeddinggemma-300m` ONNX q8（多语 0.35→0.88，须全量 re-embed）。
- 「tunnel 需手动 `create_tunnel`」→ hallway→tunnel 自动推导。
- L3 搜索须补 BM25 hybrid rerank（`_hybrid_rank` 对齐 MCP/CLI）。

**全新能力（新增段落）**
- ⭐ Living-memory 动态权重：hallways（共现自动生边）+ tunnels（跨 wing 自动晋升）+ Hebbian potentiation + Ebbinghaus decay（`hallways.py`、`dynamics.py` 新模块）。
- corpus_origin AI 语料识别 + agent persona 分类（`corpus_origin.py`，init Pass 0）。
- office 文档 ingest（PDF/docx/pptx/xlsx/EPUB via `--mode extract`）。
- 虚拟行号 + 精确 closet 指针（Tier 6a）；`wing_api` 自动路由；COCA 常用词 entity 过滤；mine 后 FTS5 校验 + repair VACUUM/rebuild。

**L1 回写**：memory-system 对比表 mempalace 列（静态图→living-memory 动态权重，是个有辨识度的新对比维度）。

**执行**：主线程写新骨架（3 个新模块段 + 改 2 处存量），1 subagent 填实现。多语 benchmark 数据如需引用以源仓 README 为准，不自行重测。

---

### B5 · deer-flow → 新（中）

**受影响页（改）**
- `prompt-system--deer-flow.md` —【中】memory/date 从 system prompt 移出，改由 `DynamicContextMiddleware` 注入为 HumanMessage `<system-reminder>`（frozen-snapshot），让 system prompt 字节恒定以打满 prefix-cache。原「memory 在 system prompt `<memory>` 块」过期。
- `mcp-skills--deer-flow.md` —【中】session pooling 改为仅 stdio（HTTP/SSE 直接 unwrapped，避免跨 anyio TaskGroup RuntimeError）；skill 存储重写为 `SkillStorage` ABC + `LocalSkillStorage` + `make_skill_tree_sandbox_readable()`；MCP extension interceptors（`extensions_config.json` 声明 callable 注入 header/auth）。原「三 transport 均支持 pooling」过期。
- `channel-remote--deer-flow.md` —【小】IM 平台 4→7（+Discord/DingTalk/WeChat，Discord mention-only + thread routing）。
- `session-recovery--deer-flow.md` / `runtime-state--deer-flow.md` —【中】新增统一持久化层（SQLAlchemy 2.0 async ORM，`RunStore` 抽象 memory/sqlite/postgres，重启从持久化恢复 run，run 创建原子化，model_name 全栈传递）。
- `query-loop--deer-flow.md` —【小】LLM circuit breaker（`llm_error_handling_middleware`）；用户取消时全 checkpoint rollback。

**新能力（暂无对应 L1）**
- Blockbuster 运行时 blocking-IO 门禁（`detect_blocking_io_strict()` pytest 钩子 + `asyncio.to_thread` 推广）。**建议折叠进 runtime-state 或 query-loop 的「可靠性」小节**，不单独建页（KB 无 reliability L1）。
- subagent per-subagent skill 加载（`SubagentConfig.skills`，None/[]/白名单）→ `multi-agent--deer-flow.md` 补一段。

**L1 回写**：prompt-system（prefix-cache 友好的 frozen system prompt 是个好对比点）、mcp-skills、session-recovery、channel-remote、multi-agent。

---

### B6 · simplemem → 0.3.0（小 · 可选）

- 现有 `_insights/simplemem--dual-storage.md` **零修改**：`MCP/` 双存储路径在打包合并 commit 中标注 unaffected，设计描述全有效。
- 唯一值得做：EvolveMem（`EvolveMem/evolvemem/`，~14000 行独立记忆进化引擎，含 KG + policy 优化 + self-upgrade + benchmark）值得新建 `_insights/simplemem--evolvemem.md`。与 dual-storage 平行、不替换它。
- 结论：**本轮可跳过**；若做，单独 1 个 insight 页即可。

---

## 4. frontmatter 基线刷新表（执行时统一更新）

| 页前缀 | 旧 source_version | 新 source_version |
|---|---|---|
| `*--agentscope.md` | `0ff492c3… 2026-04-12` | `2.0.0`（origin/HEAD `b9e36341`，2026-05-28） |
| `*--openharness.md` | `0.1.0` (2026-04-06) | `~0.1.10-dev`（origin/HEAD，2026-05-28） |
| `*--hermes-agent.md` | `0.8.0` (2026-04-08) | `0.15.0`（2026.5.28） |
| `memory-system--mempalace.md` | (2026-04-07) | `3.3.6`（2026-05-24） |
| `*--deer-flow.md` | `2.0` (2026-04-07) | origin/HEAD（2026-05-28） |
| `simplemem--dual-storage.md`（若动） | (2026-04-03) | `0.3.0`（2026-05-22） |

> 注：本地 clone 需先 `git pull` 到 origin/HEAD 再让 subagent 分析，避免读到旧代码。

---

## 5. 跨源 pattern / 新维度候选（升级中顺手识别）

这轮 delta 暴露了几个**多源同时出现**的趋势，做完上面后值得评估提炼为 `wiki/_patterns/` 或新 L1：

1. **Middleware 作为新统一扩展点**：agentscope `MiddlewareBase` 五点 / deer-flow `DynamicContextMiddleware` / hermes PluginContext 注册钩子 / deer-flow `llm_error_handling_middleware`。多家从「散落 hook」收敛到「middleware 管线」。→ `_patterns/middleware-extension-pipeline.md` 候选。
2. **工具执行安全层**：agentscope Permission Engine（.env 保护 / dangerous-path）/ hermes promptware 防御 + OSV 供应链审计 / deer-flow sandbox 文件权限。→ `_patterns/tool-execution-safety.md` 候选（横跨 tool-system + sandbox-isolation）。
3. **prefix-cache 友好的 frozen system prompt**：deer-flow DynamicContextMiddleware / hermes 跨 session prefix cache。→ context-management 或 prompt-system 设计权衡新增一节。
4. **memory 后台自动巩固**：openharness auto-dream / mempalace Hebbian+Ebbinghaus / hermes background_review。→ memory-system 设计权衡「主动 vs 被动巩固」新增一节，或 `_patterns/memory-consolidation.md`。

→ 把这些先记入 `ideas/`（status: inbox），做完 B1-B5 再回收。

---

## 6. 执行就绪检查（每批开跑前）

- [ ] 对应源仓 `git pull` 到 origin/HEAD
- [ ] 主线程写好该批大页骨架 + 该批受影响 L1 对比表的「待填列」
- [ ] subagent contract 标注 output 预算（≤15k）、surgical 约束、frontmatter 基线
- [ ] 跑完做覆盖率扫描
- [ ] 回写 L1（对比表 + 设计权衡），刷 frontmatter
- [ ] 更新 `_index.md` 与 `docs/sdk-kb-alignment.md`
