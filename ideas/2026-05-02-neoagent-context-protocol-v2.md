---
name: neoagent context protocol v2 (explicit contracts)
description: 修复版 neoagent 7 层上下文协议——把 tag 语义/字段更新策略/工具行为契约全部显式化，并解决 Layer 6/7 重叠
type: pattern
status: inbox
date: 2026-05-02
source-discussion: 2026-05-02 主线程对话（review raw/neoagent/docs/context-protocol.md）
relates_to:
  - raw/neoagent/docs/context-protocol.md  # v1 原版
  - wiki/context-management.md
  - wiki/prompt-system.md
  - wiki/memory-system.md
---

# neoagent Context Protocol v2 — 显式契约版

> v1 协议（`raw/neoagent/docs/context-protocol.md`）能跑，但靠 convention 不靠 contract。本版重点修三件事：
> 1. **静态/临时**二分错位（Layer 3 是动态的）
> 2. **Layer 6 vs Layer 7 边界不清**——它们本质不冲突（本会话压缩 vs 跨会话沉淀），但 LLM 不知道该信哪个、什么时候用哪个
> 3. **所有 XML tag 语义全靠命名暗示**——LLM 实际在猜，行为会随会话漂移

---

## 1. 核心架构（先讲清最重要的事）

**neoagent 把 agent 的所有"派生状态"全部塞进 `system` 字段；`messages` 只承载本轮活跃的原始对话。**

```
LLM 请求 = {
  system   = 7 层结构化文本（启动时静态 + 每轮重建临时）
  messages = 活跃原始对话（append-only，单一真相源）
  tools    = 工具 schema（启动时构建）
}
```

`messages` 是 source of truth，Layer 5/6/7 都是它的派生视图。任一层丢失都能从 messages 重建，反过来不行。

---

## 2. 8 层分类（三轴模型，替代"静态/临时"二分）

| 层 | 内容 | 构建时机 | 渲染格式 | 作用域 |
|----|------|---------|---------|-------|
| 1 IDENTITY | 角色身份 | 启动时 | 纯文本 | 会话 |
| 2 PERSISTENT_MEMORY | 长期事实/偏好（pre-loaded） | 启动时 | 纯文本 | 会话 |
| 3 CAPABILITIES | 已激活 skill | **skill 激活/停用时** | 纯文本 | 会话+按需 |
| 4 SECURITY | 安全约束 | 启动时 | 纯文本 | 会话 |
| 5 WORKING_MEMORY | 当前任务进展 | 每轮重建 | XML | 任务 |
| 6 COMPRESSED_HISTORY | 本会话已压缩历史 | 触发压缩时追加，每轮重渲染 | XML | 会话 |
| 7 MEMORY_CONTEXT | 跨会话检索记忆 | 每轮检索 | XML | 全局检索 |
| 8 messages | 活跃原始对话 | 每轮 append | JSON 数组 | 任务 |

**v1 → v2 关键变化**：

- 把"构建时机 / 渲染格式 / 作用域"三个独立维度拆开，Layer 3 不再被强行归为"静态"
- 加入 Layer 8（messages），让派生关系（5/6/7 ← 8）显式
- Layer 2 vs Layer 7 的区别（pre-loaded vs retrieved）通过"作用域"列点破

---

## 3. Layer 6 vs Layer 7 区别（让 LLM 清晰知道用哪个）

**它们不冲突，是两个正交维度的"过去信息"：**
- Layer 6 = **本会话** 的 **原始事件压缩**（时序、可恢复）
- Layer 7 = **跨会话** 的 **稳定知识沉淀**（抽象、不可逆）

同一事实可能在两层都出现（比如"用户偏好 TDD"，Layer 7 是 0.9 confidence 的抽象 entry，Layer 6 里是用户某轮原话的 preview）——**这不是 bug，是 feature**：一个给"什么发生了"，一个给"我们学到什么"。

### 职责矩阵

| | Layer 6 COMPRESSED_HISTORY | Layer 7 MEMORY_CONTEXT |
|---|---|---|
| **时间范围** | 仅本会话 | 跨所有会话（含本会话已沉淀部分） |
| **内容性质** | 原始事件流的结构压缩 | 已抽象的稳定知识 |
| **形态** | turn-by-turn preview + 可 recall 还原 | type/confidence 标记的 entry，不可还原原文 |
| **更新触发** | 本会话 budget 超限时 | 每轮按当前对话语义检索 |
| **置信度** | 隐含 1.0（是真实发生过的对话） | 显式 0.0-1.0（抽象过程有损） |

### LLM 该怎么用（决策契约）

| 场景 | 该信哪层 | 为什么 |
|------|---------|-------|
| "我们刚才讨论过什么？" | Layer 6（必要时 recall_turn） | 时序原文是真相 |
| "用户的长期偏好是什么？" | Layer 7 | 抽象知识就是为这设计的 |
| 两层信息看似矛盾 | **本会话 messages > Layer 6 > Layer 7** | 越靠近现在的越权威 |
| 想验证 Layer 7 entry 的来源 | 看 `source_turn` 属性 → 必要时 recall_turn | 抽象可能失真，原文可校准 |
| "这个 fact 是这次说的还是历史沉淀的？" | 看 source_turn 是否带 `@session-XXX` 后缀 | 同会话 vs 跨会话 |

### source_turn 的真正作用

不是去重，而是**溯源**：

```xml
<entry source_turn="u3@session-2026-04-15">用户偏好 TDD</entry>
<!-- ↑ 跨会话沉淀（带 session 后缀） -->

<entry source_turn="u1">v3.2a 目标：500+ tests 绿色</entry>
<!-- ↑ 本会话刚提取的（无 session 后缀，与 Layer 6 同会话） -->
```

LLM 看到 source_turn 就知道这条 Layer 7 entry 是历史沉淀还是本会话提取，能据此判断时效性和可校准性。

---

## 4. system 字段完整示例（带 SCHEMA 注释）

### Layer 1-4（静态层，纯文本拼接）

```
You are neoagent, a capable AI assistant with access to tools.
Always think step-by-step before acting.

User preferences:
- Prefer concise responses
- Always use async/await in Python code
- Project root: /Users/neo/Desktop/project/git/neoagent

## Available Skills
### code-search
Search and navigate codebases. Use Glob/Grep/Read.
### bash-execution
Execute shell commands via Bash tool.

## Security Constraints
- Do not modify system files outside project directory
- Do not expose API keys or secrets
- Confirm before destructive operations (rm, reset --hard, DROP TABLE)
```

### Layer 5 — WORKING_MEMORY（带契约头）

```xml
<working_memory version="4" at_turn="6">
  <!-- SCHEMA:
       CONSTRAINTS_AND_PREFERENCES: append-only user rules; never delete unless user retracts
       PROGRESS: overwrite each turn; 1-3 sentences
       KEY_DECISIONS: append-only with rationale; never renumber IDs
       RELEVANT_FILES: active files; prune when no longer relevant
       NEXT_STEPS: 1-3 immediate actions; remove completed
       CRITICAL_CONTEXT: overarching goal in prose; rarely changes
       ID prefix: c=constraint, d=decision, f=file, n=next-step
       Update via update_working_memory tool only -->

  <CONSTRAINTS_AND_PREFERENCES>
    - c01: 使用 pytest，TDD 优先
    - c02: async/await 核心链路，禁止同步阻塞
  </CONSTRAINTS_AND_PREFERENCES>

  <PROGRESS>
    HookManager 基础完成，pre_tool_call 已通过测试。正在写 post_tool_call。
  </PROGRESS>

  <KEY_DECISIONS>
    - d01: Hook payload 用 frozen dataclass | 防 hook 副作用污染调用链
    - d02: 4 个拦截点 (pre/post × tool/provider) | 覆盖完整生命周期
  </KEY_DECISIONS>

  <RELEVANT_FILES>
    - f01: neoagent/hooks.py (writing)
    - f02: neoagent/core/loop.py (reading)
    - f03: tests/test_hooks.py (writing)
  </RELEVANT_FILES>

  <NEXT_STEPS>
    - n01: loop.py 插入 post_tool_call 调用点
    - n02: 写 post_tool_call 的 async 测试
  </NEXT_STEPS>

  <CRITICAL_CONTEXT>
    任务：v3.2a Hooks 系统，目标 500+ tests 绿色。
  </CRITICAL_CONTEXT>
</working_memory>
```

### Layer 6 — COMPRESSED_HISTORY（带契约头）

```xml
<compressed_history>
  <!-- SCHEMA:
       SCOPE: THIS SESSION ONLY. These are raw events from the current conversation
              that were evicted from active messages due to budget pressure.
       NATURE: time-ordered preview, lossy summary, but recoverable via recall_turn.

       Distinction vs <memory-context> (Layer 7):
         - This layer = "what happened in this conversation"
         - Layer 7   = "stable knowledge across all sessions"
         - When same fact appears in both: trust this layer for raw timing/wording,
           trust Layer 7 for abstracted/distilled form.

       When to call recall_turn:
         - current task references a topic from a batch (check <turn topic>)
         - preview text feels insufficient for current decision
         - about to make a decision that may contradict a past one

       When NOT to recall:
         - speculatively for every batch (wastes context budget)
         - for batches whose topic is clearly off-task

       ID prefix: u=user, a=assistant, t=tool_use+result pair -->

  <batch id="cm_1" turns="1-3" reason="context-budget-pressure" lossy="true">
    <recoverable note="full content via recall_turn(turn_id)">
      <turn n="1" topic="hook-system-design">
        <msg id="u1" role="user" preview="设计 Hook 系统，pre/post 拦截需求" />
        <msg id="a1" role="assistant" preview="规划 4 拦截点架构方案" />
      </turn>
      <turn n="2" topic="hook-manager-impl">
        <msg id="u2" role="user" preview="实现 HookManager 基础类" />
        <msg id="t1" role="tool" preview="Read hooks.py → not found" />
        <msg id="a2" role="assistant" preview="创建 hooks.py 定义 HookManager" />
      </turn>
      <turn n="3" topic="pre-tool-call-impl">
        <msg id="u3" role="user" preview="加 pre_tool_call 拦截点" />
        <msg id="t2" role="tool" preview="Write hooks.py → 45 lines" />
        <msg id="a3" role="assistant" preview="pre_tool_call 实现完成" />
      </turn>
    </recoverable>
  </batch>
</compressed_history>
```

**v1 → v2 新增字段**：

| 字段 | 解决的问题 |
|------|----------|
| `<!-- SCHEMA -->` 头 | 一次性把 id 约定、recall 时机、lossy 性质讲清楚 |
| `reason="..."` | 告诉 LLM 为什么压缩（不是因为不重要，只是 budget） |
| `lossy="true"` | 显式标 preview 是有损的 |
| `note="..."` | 把 recoverable 的"言外之意"写出来 |
| `topic="..."` | LLM 决定是否 recall 时有判断依据 |

### Layer 7 — MEMORY_CONTEXT（带契约头）

```xml
<memory-context>
  <!-- SCHEMA:
       SCOPE: ACROSS ALL SESSIONS (including this session's already-distilled facts).
       NATURE: abstracted, stable knowledge — NOT raw events. Cannot be reverted to original wording.

       Distinction vs <compressed_history> (Layer 6):
         - This layer = "what we have learned (stable, may span months)"
         - Layer 6   = "what happened in this conversation (raw, lossy preview)"
         - source_turn attribute tells you the origin:
             * "u3@session-2026-04-15" → distilled from a past session
             * "u3" (no @session)      → distilled from this session's earlier turns

       Trust order when info conflicts:
         active messages > Layer 6 (recent raw) > Layer 7 (older abstract)
       If contradicted by messages: flag in PROGRESS, don't blindly trust this layer.
       If confidence < 0.7: treat with caution; verify via recall_turn if source is local.

       Fields:
         type: preference | fact | goal | event | skill
         confidence: 0.0-1.0 (your trust in this entry)
         source_turn: traceability marker (use for recall_turn if local) -->

  <entry type="preference" confidence="0.9" source_turn="u3@session-2026-04-15">
    用户偏好 TDD，每个功能先写测试再实现
  </entry>
  <entry type="fact" confidence="0.85" source_turn="u1@session-2026-04-10">
    neoagent 项目路径：/Users/neo/Desktop/project/git/neoagent
  </entry>
  <entry type="goal" confidence="0.95" source_turn="u1@session-2026-05-02">
    v3.2a 目标：Hooks + MCP，保持 500+ tests 绿色
  </entry>
</memory-context>
```

**v1 → v2 新增字段**：`source_turn` 让 Layer 6/7 去重成为可能；契约头明确"messages 优先于 memory"的冲突解决规则。

---

## 5. messages 数组（原始事实层）

```json
[
  {"role": "user", "content": "好，pre_tool_call 看起来不错。现在加上 post_tool_call。"},
  {"role": "assistant", "content": [
    {"type": "text", "text": "先看下 loop.py 里 tool_call 执行位置。"},
    {"type": "tool_use", "id": "toolu_05", "name": "Read",
     "input": {"file_path": "neoagent/core/loop.py"}}
  ]},
  {"role": "user", "content": [
    {"type": "tool_result", "tool_use_id": "toolu_05",
     "content": "async def execute_tool(self, call):\n    payload = ToolCallPayload(...)\n    ..."}
  ]}
]
```

### tool_result 被 free 后的形态（v2 改进）

```json
{
  "type": "tool_result",
  "tool_use_id": "toolu_05",
  "content": "[FREED:toolu_05 size=2.3KB | recall_tool_result(toolu_05) to restore | freed_reason=large-file-no-longer-active]"
}
```

`freed_reason` 让 LLM 能判断该不该 recall，而不是只看到一个空洞的占位符。

---

## 6. 工具契约（操作动词的语义闸口）

XML 层的 schema 注释 + 工具 description 的契约 = 双重闸口。LLM 写入时必看工具 description，读取时必看 schema 注释。

### update_working_memory

```json
{
  "name": "update_working_memory",
  "description": "Update a single working_memory field. Each field has strict semantics:\n\n- CONSTRAINTS_AND_PREFERENCES: append-only user rules. Never delete unless user explicitly retracts. Operation: append only.\n- PROGRESS: current task status, 1-3 sentences. Operation: overwrite only.\n- KEY_DECISIONS: append-only with rationale. Format: '{id}: {decision} | {rationale}'. Operation: append only; never renumber.\n- RELEVANT_FILES: active files. Operation: append/remove. Prune when no longer relevant.\n- NEXT_STEPS: 1-3 immediate actions. Operation: append/remove. Remove completed items.\n- CRITICAL_CONTEXT: overarching goal in prose. Operation: overwrite only; rarely changes.\n\nID conventions: c01-c99 (constraints), d01-d99 (decisions), f01-f99 (files), n01-n99 (next-steps). IDs are stable; never reassign.",
  "input_schema": {
    "type": "object",
    "properties": {
      "field": {"enum": ["CONSTRAINTS_AND_PREFERENCES", "PROGRESS", "KEY_DECISIONS",
                         "RELEVANT_FILES", "NEXT_STEPS", "CRITICAL_CONTEXT"]},
      "operation": {"enum": ["append", "overwrite", "remove"]},
      "id": {"type": "string", "description": "Required for append/remove on list fields"},
      "content": {"type": "string"}
    },
    "required": ["field", "operation"]
  }
}
```

### recall_turn

```json
{
  "name": "recall_turn",
  "description": "Restore the full original messages of a compressed turn. Use when:\n- Current task references a topic from a compressed batch (check <turn topic> attribute)\n- Preview text feels insufficient for current decision\n- About to make a decision that may contradict past decisions and need to verify\n\nDo NOT use:\n- Speculatively for every batch (defeats compression)\n- For batches whose topic is clearly off-task\n\nRecalled content is re-injected into the next message and counts against context budget.",
  "input_schema": {
    "type": "object",
    "properties": {
      "turn_id": {"type": "string", "description": "e.g. 'u1', 'a2', 't3'"}
    },
    "required": ["turn_id"]
  }
}
```

### free_tool_result

```json
{
  "name": "free_tool_result",
  "description": "Free a large tool_result from messages, replacing with a placeholder. Original preserved in session, restorable via recall_tool_result. Use when:\n- A tool returned >2KB content\n- The content was used and is no longer actively referenced\n- Context budget is tight\n\nThe placeholder includes size and freed_reason to help future-you decide whether to recall.",
  "input_schema": {
    "type": "object",
    "properties": {
      "tool_use_id": {"type": "string"},
      "freed_reason": {"type": "string",
                       "description": "Brief: e.g., 'large-file-no-longer-active', 'one-shot-lookup-done'"}
    },
    "required": ["tool_use_id", "freed_reason"]
  }
}
```

---

## 7. 上下文组装流程

```
每轮 LLM 调用前：

1. msgs_for_llm = messages 数组截断（超 budget×70% 的旧 turns 进入压缩队列）
2. system_static = build_static_layers()                # Layer 1-4
3. wm_xml       = render_working_memory(session.wm)      # Layer 5
4. ch_xml       = render_compressed_history(batches)     # Layer 6（已剔除被 promote 到 Layer 7 的 turn）
5. mc_xml       = render_memory_context(retrieve(...))   # Layer 7（按 messages 检索 top-k）
6. system = system_static + "\n\n" + wm_xml + "\n\n" + ch_xml + "\n\n" + mc_xml
7. → provider.call(system, msgs_for_llm, tools)
```

---

## 8. v1 → v2 改动一览

| 改动 | 解决的问题 |
|------|----------|
| 三轴分类替代静态/临时二分 | Layer 3 不再格格不入 |
| 加入 Layer 8（messages）显式入图 | 派生关系（5/6/7 ← 8）一目了然 |
| 每个 XML 块加 `<!-- SCHEMA -->` 头 | tag 语义显式，避免 LLM 跨轮漂移 |
| working_memory 字段拆为独立 sub-tag | 支持 append/overwrite/remove 区分操作语义 |
| compressed_history 加 topic / lossy / reason | LLM 能判断该不该 recall |
| compressed_history SCHEMA 头明确"本会话 only" | LLM 不会和 Layer 7 混用 |
| memory-context SCHEMA 头明确"跨会话稳定知识" + 信任顺序 | 矛盾时 LLM 知道该信谁 |
| memory-context 加 source_turn（本/跨会话标记）| 让 LLM 区分本会话沉淀 vs 历史沉淀，并支持溯源 |
| 工具 description 写死操作契约 | LLM 写入时有 schema 闸口 |
| free_tool_result 加 freed_reason | LLM 能判断该不该 recall_tool_result |

---

## 9. 这次没解决的事（留给 v3）

- **batch 边界的语义切分**：现在按 budget 触发的 token 切分，topic 标签靠 LLM 摘要时填，质量取决于 LLM。理想是按"主题转换"切分
- **本会话 Layer 7 entry 的实时性**：MemoryManager 在本会话中提取的 entry，其 source_turn 还在 Layer 6 里——LLM 可能两边都看到（这没错，是 feature），但需要 eval 验证它真的会用 Layer 6 校准 Layer 7，而不是单纯被双注入困惑
- **跨模型可移植性**：v2 让协议变得更显式但还是 XML-in-system 的 Anthropic-friendly 形态。OpenAI 系更倾向把状态放 messages

---

## 10. 升级路径建议

如果要在 neoagent v3.3 落地 v2，建议分阶段：

**阶段 1（无破坏）**：所有 XML 块加 SCHEMA 注释 + 工具 description 加契约。这一步纯文本变化，对现有调用零影响。

**阶段 2（小破坏）**：working_memory 字段从 plain text 块改成嵌套 sub-tag。需要更新 `update_working_memory` 工具实现。

**阶段 3（溯源）**：MemoryManager 写入 entry 时记录 source_turn（本会话 turn id 或 `tid@session-YYYY-MM-DD`）。这一步**不是为了去重**，而是为了让 LLM 知道一条 Layer 7 entry 来自本会话还是跨会话历史，并支持 recall_turn 校准。

**阶段 4（评估）**：跑一组 eval（建议在 wiki/evaluation-observability 里建 baseline），对比 v1 vs v2 在长会话下的：
- 字段使用一致性（CONSTRAINTS 是不是真的 append-only）
- Layer 6/7 区分清晰度（LLM 在矛盾场景下是否按"messages > Layer 6 > Layer 7"信任顺序决策）
- recall_turn 调用合理性（是否 speculative recall 减少了，按 topic 触发的精准 recall 增加了）

---

## Status

`status: inbox` — 待 review 决定升级路径：

- 升级到 `wiki/_patterns/explicit-context-schema.md`（如果可被其他 agent 框架借鉴）
- 或升级到 `wiki/_insights/neoagent--context-protocol-v2.md`（如果只对 neoagent 适用）
- 或落到 `projects/neoagent/decisions/YYYY-MM-DD-context-protocol-v2.md`（如果决定真在 v3.3 落地）
