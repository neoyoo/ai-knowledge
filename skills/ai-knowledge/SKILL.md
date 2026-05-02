---
name: ai-knowledge
description: |
  专业智能体开发设计知识库——在做以下事情时**必须首先查它**（优先于 WebSearch、派 subagent 调研、grep 源码）：
  
  - 设计 agent 架构 / 选型（loop / 沙箱 / 内存 / 多 agent / MCP / channel / session / 工具系统 等）
  - 对比具体项目做法（Claude Code / AgentScope / DeerFlow / OpenHarness / Hermes-Agent / Modal / E2B / neoagent 等）
  - 写 tool / skill / prompt（找模板、anti-pattern、跨项目对比）
  - 组合两个及以上概念解决问题（查 wiki/_patterns/ 跨概念组合模式）
  - 蒸馏闪念 / 评估想法落地可能（查 ideas/ inbox，含 status: inbox/incubating/promoted/dead）
  - 回看真实项目决策（查 projects/<proj>/decisions/）
  
  典型触发短语：「怎么设计…」「选哪个方案」「有没有现成 pattern」「XX 项目是怎么做…」「类似的设计别人怎么做」「把 A 和 B 组合起来」「这个思路值得做吗」「为什么这样设计」「怎么避免 X 陷阱」。
  
  **找不到对应内容时必须把摩擦记入 `projects/<proj>/kb-friction.md`——这是 KB 自我进化的真实输入。不记等于 KB 继续死着。**
---

# AI Knowledge Navigator

> 专业智能体开发设计知识库。wiki/ 解答「为什么这样设计」，cookbook/ 解答「怎么动手做」，ideas/ 收容「还没定型的好想法」，projects/ 记录「我们真正做过的决策」，shelf/ 归档「不够成熟但有亮点的源」。

## 使用纪律（最重要）

1. **先查 KB，后去外面** — 碰到设计/选型/对比任务，先用 Glob/Grep/Read 翻 KB。只有 KB 明确没有时，才派 subagent 调研或 WebSearch
2. **摩擦必记** — 每次查询"找不到 / 答非所问 / 不够深"的情况，**立刻**追加一条到 `/Users/neo/Desktop/project/git/ai-knowledge/projects/<proj>/kb-friction.md`。格式见下方"摩擦日志格式"
3. **真决策要落档** — 重大决策（选型、架构、工具换底等）做完写 `projects/<proj>/decisions/YYYY-MM-DD-<slug>.md`，引用支撑该决策的 KB 页
4. **别自己写内容到 wiki/** — skill 只负责查和记摩擦。往 wiki/ 写东西是 kb-ingest 和 kb-maintain 的职责

## 知识库位置

```
/Users/neo/Desktop/project/git/ai-knowledge/       ← 路径含空格
├── wiki/           L1 概念 + L2 实现 + 洞察 + 组合模式
│   ├── *.md        15 个 L1 架构概念页
│   ├── _impl/      per-source L2 实现分析（<concept>--<source>.md）
│   ├── _insights/  单源精彩设计抽出（<source>--<design>.md）
│   └── _patterns/  跨概念组合模式（可迁移架构语汇）
├── cookbook/       实操参考（prompts/tools/skills × patterns/templates/anti-patterns）
├── ideas/          想法 inbox（status: inbox/incubating/promoted/dead）
├── shelf/          归档源（SHELF.md 有候选提升项）
├── projects/       真实项目决策和摩擦日志
├── schema/         规则与模板（ontology / lint-rules / ingest-strategies / page-templates）
└── docs/           KB 元文档（sdk-kb-alignment 等）
```

## 查询方法（按优先级）

1. **走常见任务表**（本页下方）快速路由到具体页
2. **查索引** — 读 `_index.md`：
   - `wiki/_index.md` 总导航
   - `wiki/_patterns/_index.md` 组合模式
   - `cookbook/_index.md`、`cookbook/*/_index.md` 实操层
   - `ideas/_index.md` 想法状态一览
3. **精确搜** — 用 Grep tool（不是 bash grep）：
   ```
   Grep(pattern="关键词", path="/Users/neo/Desktop/project/git/ai-knowledge/wiki")
   Grep(pattern="关键词", path="/Users/neo/Desktop/project/git/ai-knowledge/cookbook")
   Grep(pattern="关键词", path="/Users/neo/Desktop/project/git/ai-knowledge/ideas")
   Grep(pattern="关键词", path="/Users/neo/Desktop/project/git/ai-knowledge/shelf")
   ```
4. **用 Read 读完整页面**——不要猜内容，直接读

---

## Wiki 导航（L1 架构原理层）

| 概念 | 文件 | 关键词 |
|------|------|--------|
| Agent 主循环 | `wiki/query-loop.md` | loop, state machine, turns, agentic loop |
| Prompt 动态组装 | `wiki/prompt-system.md` | system prompt, dynamic sections, priority |
| 工具调用系统 | `wiki/tool-system.md` | tool, function calling, dispatch, executor, registry |
| Context 窗口管理 | `wiki/context-management.md` | context, compression, window, truncation, freed-recall |
| 跨会话记忆 | `wiki/memory-system.md` | memory, persistent, cross-session |
| 运行时状态 | `wiki/runtime-state.md` | state, session state, checkpoint |
| 会话恢复 | `wiki/session-recovery.md` | recovery, resume, crash, fault tolerance |
| 多 Agent 协作 | `wiki/multi-agent.md` | orchestrator, worker, delegation, task envelope |
| 事件 Hooks | `wiki/hooks.md` | hook, event, lifecycle, pre/post tool, interceptor |
| MCP / Skills 扩展 | `wiki/mcp-skills.md` | MCP, skill, extension, plugin |
| 远程渠道接入 | `wiki/channel-remote.md` | channel, API, SSE, FastAPI, HTTP streaming |
| 评估与可观测性 | `wiki/evaluation-observability.md` | eval, metric, logging, trace |
| **沙箱隔离**（新）| `wiki/sandbox-isolation.md` | sandbox, gVisor, Firecracker, Docker, isolation, threat model |
| **微调系统**（新）| `wiki/finetuning-system.md` | finetuning, 微调, SFT, DPO, GRPO, RL, training pipeline, prompt optimization |
| **Agent 注册发现**（新）| `wiki/agent-registry-discovery.md` | registry, discovery, AgentCard, Nacos, A2A, service discovery, 注册中心 |

## Wiki 组合模式（`wiki/_patterns/`，跨概念）

查"如何把 A 概念 + B 概念组合起来解决 X 问题"类的需求：

| Pattern | 文件 | 涉及概念 |
|---------|------|---------|
| 工具元数据驱动的上下文生命周期（free/recall + system 注入 + events） | `wiki/_patterns/tool-metadata-driven-context-lifecycle.md` | tool-system + context-management + prompt-system + hooks |

组合模式的选入标准：跨 ≥2 个 L1 概念 + 可迁移 + 有参考实现。

## Wiki 单源洞察（`wiki/_insights/`）

单个项目里**同类源未见**的精彩设计。按需检索：

```
Glob("wiki/_insights/*.md")
```

---

## Cookbook 导航（实操参考层）

### Prompts

| 任务 | 路径 |
|------|------|
| 浏览所有 prompt pattern | `cookbook/prompts/patterns/_index.md` |
| 推理增强（CoT / ToT / Self-Consistency）| `cookbook/prompts/patterns/chain-of-thought.md` |
| 行动循环（ReAct / Reflexion / ART） | `cookbook/prompts/patterns/react.md` |
| 可复制模板 | `cookbook/prompts/templates/_index.md` |
| 避免常见 prompt 错误 | `cookbook/prompts/anti-patterns/_index.md` |

### Tools

| 任务 | 路径 |
|------|------|
| 工具定义总览 | `cookbook/tools/definitions/_index.md` |
| 文件操作 | `cookbook/tools/definitions/file-operations.md` |
| Shell 执行 | `cookbook/tools/definitions/shell-execution.md` |
| Web 工具 | `cookbook/tools/definitions/web-tools.md` |
| Agent 编排 | `cookbook/tools/definitions/agent-orchestration.md` |
| MCP 集成 | `cookbook/tools/definitions/mcp-integration.md` |
| 工具设计原则 | `cookbook/tools/patterns/tool-design-principles.md` |
| 错误处理 | `cookbook/tools/patterns/error-handling.md` |
| 权限模型 | `cookbook/tools/patterns/permission-model.md` |

### Skills

| 任务 | 路径 |
|------|------|
| 选 skill 类型 | `cookbook/skills/patterns/_index.md` |
| 复制 skill 模板 | `cookbook/skills/templates/_index.md` |
| 避免 skill 设计错误 | `cookbook/skills/anti-patterns/_index.md` |

---

## Ideas / Shelf / Projects 导航

### `ideas/`（想法 inbox）

评估某个想法是否已经有人提过、是否正在孵化、已落地还是已死亡：

```
Read("/Users/neo/Desktop/project/git/ai-knowledge/ideas/_index.md")
```

idea 状态流转：`inbox → incubating → promoted`（进 wiki/_patterns/ 或 _insights/ 或 practice/）或 `dead`（保留死亡理由）。

### `shelf/`（归档源的候选提升项）

shelf 项目虽不合并 wiki/，但每个 `shelf/<src>/SHELF.md` 会列出**候选提升项**——这些是可以单独抽到 wiki/_insights/ 或 wiki/_patterns/ 的神器设计。

```
Read("/Users/neo/Desktop/project/git/ai-knowledge/shelf/neoagent/SHELF.md")
Read("/Users/neo/Desktop/project/git/ai-knowledge/shelf/simplemem/SHELF.md")
```

### `projects/`（真实决策 + 摩擦日志）

- `projects/<proj>/decisions/YYYY-MM-DD-<slug>.md` — 过往决策 + 依据的 KB 页
- `projects/<proj>/kb-friction.md` — 使用 KB 的摩擦实时日志（标准结构；不存在时按需创建）
- `projects/<proj>/open-questions/` — 尚未回答的问题（可选结构）

查相似场景的过往决策：

```
Glob("projects/*/decisions/*.md")
```

---

## 常见 Agent 开发任务 → 页面

| 我想… | 先读（why） | 再读（how） |
|-------|-----------|-----------|
| 设计 agent 主循环 | `wiki/query-loop.md` | `cookbook/prompts/patterns/react.md` |
| 写 agent system prompt | `wiki/prompt-system.md` | `cookbook/prompts/templates/` |
| 实现工具调用 | `wiki/tool-system.md` | `cookbook/tools/definitions/_index.md` |
| 给 agent 加跨会话记忆 | `wiki/memory-system.md` | — |
| 设计多 agent 编排 | `wiki/multi-agent.md` | `cookbook/prompts/patterns/prompt-chaining.md` |
| 添加事件 Hook | `wiki/hooks.md` | `cookbook/tools/definitions/agent-orchestration.md` |
| 集成 MCP 工具 | `wiki/mcp-skills.md` | `cookbook/tools/definitions/mcp-integration.md` |
| 接入 API / HTTP 渠道 | `wiki/channel-remote.md` | — |
| 管理长对话 context | `wiki/context-management.md` | `cookbook/prompts/patterns/rag.md` |
| **tool_result 过大想折叠可召回** | `wiki/_patterns/tool-metadata-driven-context-lifecycle.md` | shelf/neoagent 参考实现 |
| 实现断点续传 | `wiki/session-recovery.md` | — |
| **Web 服务化选沙箱** | `wiki/sandbox-isolation.md`（含 gVisor 专题）| — |
| **设计 agent 注册/服务发现机制** | `wiki/agent-registry-discovery.md`（含 Java 后端类比 + 5 家对比） | — |
| **分布式 multi-agent 选型** | `wiki/multi-agent.md`（含分布式就绪度行）| `wiki/agent-registry-discovery.md` |
| **评估某个想法** | `ideas/_index.md`（是否已有条目）| 若新想法 → 写入 ideas/inbox |
| **回看类似决策怎么做的** | `projects/*/decisions/` | — |
| 选推理策略 | `cookbook/prompts/patterns/_index.md` | `cookbook/prompts/patterns/chain-of-thought.md` |
| 设计和写 skill | `cookbook/skills/patterns/_index.md` | `cookbook/skills/templates/_index.md` |
| 避免 prompt 反模式 | `cookbook/prompts/anti-patterns/_index.md` | — |
| 评估 agent 质量 | `wiki/evaluation-observability.md` | — |

---

## 摩擦日志格式（严格遵守）

每次 KB 查询结束，如遇"找不到 / 答非所问 / 不够深"，在 `projects/<proj>/kb-friction.md` 追加：

```markdown
| 时间 | 查询 | 去哪 | 命中 | 摩擦 / 缺什么 |
|------|-----|------|------|--------------|
| 2026-04-19 12:47 | "多用户 Web 服务 + 沙箱总体架构" | `wiki/sandbox-isolation.md` | ✅ 好 | 威胁模型分档清晰 |
| 2026-04-19 12:50 | "文件下载的安全问题" | 没独立页 | ❌ 严重缺口 | 需要 download-safety 专题 |
```

项目名 `<proj>` 从上下文推断（比如 `trip-os`）。如不确定问用户。

---

## 升级产出的流向

发现新想法 / 新模式时，**不要直接写 wiki/**，而是：

- 一条想法 → `ideas/YYYY-MM-DD-<slug>.md`（status: inbox）
- 跨概念组合模式（成熟的）→ 由 kb-maintain / kb-capture-practice 审过再升级到 `wiki/_patterns/`
- 单源精彩设计 → 同上，升级到 `wiki/_insights/`
- 真实决策 → `projects/<proj>/decisions/YYYY-MM-DD-<slug>.md`

skill 本身是只读+记摩擦，**不动 wiki 主干**。
