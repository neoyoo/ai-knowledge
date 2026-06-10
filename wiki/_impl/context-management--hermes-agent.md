---
title: "Context Management — Hermes Agent"
category: L2
parent: "[[context-management]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 的上下文管理分为两条完全独立的流水线：**对话时压缩**（`ContextCompressor`，运行时在线）和**轨迹后压缩**（`TrajectoryCompressor`，离线批处理）。前者服务于 agent 长跑会话的上下文续存，后者服务于 RL/训练数据的 token 预算控制。两者共享"头尾保护 + 中间摘要替换"的核心思路，但触发机制、保护策略、摘要质量目标截然不同。

与 Claude Code 的"状态快照投影"理念不同，Hermes 的压缩更接近"信息密度蒸馏"：通过结构化摘要模板（Goal / Progress / Decisions / Files / Next Steps）和跨压缩迭代更新机制，尽量让后续 agent 能无缝衔接前序工作，而不是简单截断。

## 架构分析

### 一、双流水线架构

```
会话时（在线）                      训练后处理（离线）
──────────────────────             ──────────────────────────
ContextCompressor                  TrajectoryCompressor
  agent/context_compressor.py        trajectory_compressor.py
  触发：token >= 50% context limit   触发：轨迹 token > target_max_tokens
  输入：OpenAI messages 格式         输入：JSONL conversations 格式
  保护：token-budget 尾部保护        保护：按角色 first-N + last-N turns
  摘要：7-段结构化模板               摘要：中性视角行动描述（750 tokens 目标）
  迭代：跨压缩复用前次摘要           迭代：无（每次独立生成）
  并发：同步                         并发：asyncio + Semaphore（50并发）
```

### 二、对话时压缩：ContextCompressor

**文件**：`agent/context_compressor.py`

#### 初始化与阈值计算

`ContextCompressor.__init__()` 在 agent 启动时由 `run_agent.py` 实例化，关键参数均读自 `config.yaml` 的 `compression` 节：

```python
# run_agent.py ~L1093
compression_threshold = float(_compression_cfg.get("threshold", 0.50))  # 50%
compression_target_ratio = float(_compression_cfg.get("target_ratio", 0.20))  # 20%
compression_protect_last = int(_compression_cfg.get("protect_last_n", 20))
compression_summary_model = _compression_cfg.get("summary_model") or None
```

`context_length` 通过 `get_model_context_length()` 多级解析（见"Token 计数"章节），`threshold_tokens = context_length * threshold_percent`。

尾部保护的 token budget 用同一比例推算：

```python
target_tokens = int(threshold_tokens * summary_target_ratio)  # 默认 0.20
self.tail_token_budget = target_tokens  # 约 10% context_length
self.max_summary_tokens = min(int(context_length * 0.05), 12_000)
```

#### 触发机制：双路检测

Hermes 在两处触发压缩：

1. **预飞检查（preflight）**：进入主 loop 前调用 `should_compress_preflight(messages)` 粗估 token，处理"切换到小窗口模型"等场景（`run_agent.py ~L7000`）。
2. **每轮 API 返回后**：`should_compress(last_prompt_tokens)` 精确检查。若 `last_prompt_tokens == 0`（API 断连等异常），回退到 `estimate_messages_tokens_rough()` 粗估，避免无限增长（注释 `#2153`）。

```python
# run_agent.py ~L8858
if self.compression_enabled and _compressor.should_compress(_real_tokens):
    messages, active_system_prompt = self._compress_context(...)
```

#### 五步压缩算法

`compress()` 方法（`context_compressor.py ~L565`）：

**第一步：工具输出剪枝（无 LLM 调用）**

`_prune_old_tool_results()` 把尾部 `protect_last_n * 3` 之外的所有 `role=tool` 消息内容（>200 chars）替换为占位符 `[Old tool output cleared to save context space]`。这是一个纯字符串操作的廉价预处理，能快速释放大量 token（工具调用返回往往是最占 token 的部分）。

**第二步：头部保护**

`compress_start = protect_first_n`（默认 3，覆盖系统 prompt + 第一轮用户/助手交互）。若边界落在 `role=tool` 消息上，`_align_boundary_forward()` 向后滑动，避免切割工具调用组。

**第三步：尾部保护（token-budget 策略）**

`_find_tail_cut_by_tokens()` 从消息末尾向前反向扫描，累计 token 直到超过 `tail_token_budget`（默认约 10% 上下文窗口），此处为保护 token 量而非固定消息数。同时对 `protect_last_n`（默认 20）做兜底，取二者保护范围的最大值。边界落在工具调用组中间时，`_align_boundary_backward()` 把边界拉回到该组的父 assistant 消息之前，防止孤儿 tool result。

**第四步：中间段结构化摘要**

`_generate_summary()` 将中间消息序列化为 `_serialize_for_summary()` 格式（每条最多 3000 chars，tool_result 保留 3000 chars，assistant tool_calls 参数保留 500 chars），然后调用辅助 LLM。

摘要分两种模式：
- **首次压缩**：从零生成，使用 7 段结构化模板（Goal / Constraints & Preferences / Progress[Done/InProgress/Blocked] / Key Decisions / Relevant Files / Next Steps / Critical Context）。
- **再次压缩**（`_previous_summary` 不为空）：迭代更新，保留前次摘要中仍有效的信息，把"In Progress"项推进到"Done"，追加新进展。这是 Hermes 区别于其他系统的核心设计点——跨多次压缩的知识积累不被清零。

摘要 token budget 动态计算：`max(2000, min(content_tokens * 0.20, max_summary_tokens))`，大窗口模型自动获得更丰富的摘要，而非被硬性 8K 封顶。

**第五步：组装 + 工具对完整性修复**

把 head + 摘要消息 + tail 拼接。摘要消息的 role 经过智能选择（避免与相邻消息形成相同 role 的连续序列；无法回避时，直接合并到 tail 第一条消息）。

最后 `_sanitize_tool_pairs()` 修复两类孤儿问题：
1. tool result 引用了已被压缩掉的 tool_call → 删除孤儿 result
2. assistant 的 tool_calls 丢失了对应 result → 插入 stub result `[Result from earlier conversation — see context summary above]`

#### 压缩上下文：_compress_context 的职责扩展

`run_agent.py` 的 `_compress_context()` 在调用 `context_compressor.compress()` 前后做了更多工作：
- 调用 `flush_memories()` 和 `memory_manager.on_pre_compress()` 确保记忆在压缩前持久化
- 压缩后注入 todo 快照（todo_store）
- 在 SQLite session DB 中结束旧 session、创建新 session（continuation lineage）
- 重置文件读取去重缓存（避免压缩后工具 stub 替代了真实文件内容）

#### 失败冷却与模型动态更新

摘要生成失败后进入 600 秒冷却（`_SUMMARY_FAILURE_COOLDOWN_SECONDS = 600`），冷却期间压缩仍执行但不生成摘要（中间段被静默删除）。与 Claude Code 的"熔断"不同，Hermes 不停止压缩本身，只停止摘要生成。

模型切换（`/model` 命令）后，`switch_model()` 同步更新 `context_compressor.model/context_length/threshold_tokens`，确保压缩阈值随模型实时变化。

### 三、后处理压缩：TrajectoryCompressor

**文件**：`trajectory_compressor.py`

#### 轨迹格式

输入是 JSONL 文件，每行一条轨迹，字段 `conversations` 是消息数组，每条消息格式为 `{"from": "system"/"human"/"gpt"/"tool", "value": "..."}` —— 与对话时压缩的 OpenAI messages 格式不同。

#### 保护策略

按角色首次出现划定头部保护区：`protect_first_system` + `protect_first_human` + `protect_first_gpt` + `protect_first_tool`（均可配置）。尾部保护固定最后 N 轮（默认 4）。`_find_protected_indices()` 用集合存储被保护的 turn 索引。

#### 精确 token 计数 vs 粗估

与对话时压缩的粗估（4 chars/token）不同，后处理压缩使用真实 tokenizer（默认 `moonshotai/Kimi-K2-Thinking`，可配置）：`AutoTokenizer.from_pretrained()` + `tokenizer.encode()`。这是因为训练数据的 token 预算控制要求精确，误差不可接受。

#### 最小压缩原则

只压缩"恰好够用"的内容。算法计算 `tokens_to_save = total_tokens - target_max_tokens`，从可压缩区域起始向后累积，累计到 `tokens_to_save + summary_target_tokens`（默认 750）时停止，剩余中间段保持不变。这与对话时压缩"压缩所有中间段"的策略形成对比。

```python
# trajectory_compressor.py ~L712
target_tokens_to_compress = tokens_to_save + self.config.summary_target_tokens
for i in range(compress_start, compress_end):
    accumulated_tokens += turn_tokens[i]
    compress_until = i + 1
    if accumulated_tokens >= target_tokens_to_compress:
        break
```

#### 并发异步处理

`compress_trajectory_async()` + `asyncio.Semaphore(max_concurrent_requests=50)` 支持目录级批量并发压缩，配合 `rich.progress` 显示进度。摘要失败有 fallback 文本，不阻断批量流程。

#### 输出格式

压缩后的轨迹保留原始 JSONL 结构，将被压缩的多条消息替换为单条 `{"from": "human", "value": "[CONTEXT SUMMARY]: ..."}` 消息。系统 prompt 追加 summary notice 文本。每条被压缩的轨迹可附加 `compression_metrics` 字段（`was_compressed`、`compression_ratio`、`tokens_saved` 等），供训练管道质量分析。

### 四、Token 计数：多级上下文长度解析

**文件**：`agent/model_metadata.py`

#### 粗估函数

`estimate_tokens_rough(text)` = `len(text) // 4`（4 chars/token）
`estimate_messages_tokens_rough(messages)` = `sum(len(str(msg)) for msg in messages) // 4`
`estimate_request_tokens_rough(messages, system_prompt, tools)` —— 将 tools schema 纳入估算，补了一个重要盲点：50+ 工具的 schema 序列化可额外消耗 20-30K token。

#### 上下文长度解析优先级（`get_model_context_length()`，10 级）

1. 用户 config.yaml 显式配置（最高优先）
2. 持久缓存（`~/.hermes/context_length_cache.yaml`，key 格式 `model@base_url`）
3. 自定义 endpoint `/models` 接口
4. 本地服务器直接查询（Ollama `/api/show`、LM Studio `/api/v1/models`、vLLM `/version`、llama.cpp `/v1/props`）
5. Anthropic `/v1/models` API（仅常规 API key，OAuth 不支持）
6. 提供商感知查询（Nous 通过 OpenRouter suffix 匹配；其他通过 models.dev）
7. OpenRouter 实时 API 元数据
8. 硬编码默认值（宽泛的模型家族模式，最长优先匹配）
9. 本地服务器兜底查询
10. 默认 128K

#### 上下文探测（Context Probing）

当 API 返回上下文超限错误时，`parse_context_limit_from_error()` 用正则解析错误文本中的实际限制数字；若解析失败，`get_next_probe_tier()` 按 `[128K → 64K → 32K → 16K → 8K]` 逐级降档。探测到的真实限制通过 `save_context_length()` 持久化，下次会话直接使用，不再探测。

### 五、@-引用展开协议（context_references.py）

**文件**：`agent/context_references.py`

用户在消息中写入 `@` 前缀的引用符号时，Hermes 在将消息发送给 LLM 之前，先同步展开所有引用并将内容附加到消息末尾。这是一种"用户侧上下文注入"机制，与系统 prompt 注入或工具调用完全独立。

#### 支持的引用语法

| 引用符号 | 含义 | 展开结果 |
|--------|------|--------|
| `@diff` | 当前工作区未暂存的 diff | `git diff` 输出，代码块包裹 |
| `@staged` | 已暂存但未提交的 diff | `git diff --staged` 输出 |
| `@file:path` | 文件全文 | 带语言标注的代码块 |
| `@file:path:10-20` | 文件行范围（10-20 行） | 仅返回指定行区间 |
| `@folder:path` | 目录树列表 | 文件/子目录树形结构，优先用 `rg --files` 生成 |
| `@git:N` | 最近 N 条提交的 diff（N 最大 10） | `git log -N -p` 输出 |
| `@url:value` | 抓取 URL 并提取正文 | `web_extract_tool` Markdown 格式 |

正则 `REFERENCE_PATTERN` 使用 negative lookbehind `(?<![\w/])` 避免误匹配路径中间的 `@`（如邮箱地址）。行范围语法由 `@file:path` 的专属二次解析完成：`re.match(r"^(?P<path>.+?):(?P<start>\d+)(?:-(?P<end>\d+))?$")`，单行时 `end` 默认等于 `start`。

#### 敏感路径拦截

`_ensure_reference_path_allowed()` 在展开任何文件/目录引用前做双重校验：

1. **精确文件黑名单**：`~/.ssh/id_rsa`、`~/.ssh/id_ed25519`、`~/.ssh/config`、`~/.ssh/authorized_keys`、`~/.netrc`、`~/.pgpass`、`~/.npmrc`、`~/.pypirc`、`~/.bashrc`、`~/.zshrc`、`~/.profile` 等常见凭证文件，以及 Hermes 内部的 `$HERMES_HOME/.env`。
2. **目录前缀黑名单**：`~/.ssh`、`~/.aws`、`~/.gnupg`、`~/.kube`、`~/.docker`、`~/.azure`、`~/.config/gh`、`$HERMES_HOME/skills/.hub`（内部 skill hub 目录）。

任何被拦截的引用都以 warning 形式返回给用户，而不是静默失败。

此外，路径解析时强制检查 `allowed_root`（默认等于 `cwd`）：相对路径解析后必须在 `allowed_root` 内，否则抛出 `"path is outside the allowed workspace"` 异常，防止 `@file:../../secret` 形式的工作区逃逸。

#### Token 注入追踪与限额

展开后所有 block 的粗估 token 累加到 `injected_tokens`（`estimate_tokens_rough(block)`）：

- **硬限制**：`injected_tokens > context_length * 0.50` → 拒绝展开，`ContextReferenceResult.blocked = True`，消息保持原样发送
- **软警告**：超过 `context_length * 0.25` 时追加 warning 文本，但不阻断

最终消息结构：原始消息去掉所有 `@token`（`_remove_reference_tokens()` 做空白归一化）+ `--- Context Warnings ---` 节（如有）+ `--- Attached Context ---` 节（每个展开块带 emoji 标签和 token 计数前缀）。

`ContextReferenceResult` 数据类保存完整字段：`message`（展开后）、`original_message`（展开前）、`references`（解析到的所有引用）、`warnings`、`injected_tokens`、`expanded`、`blocked`，供上层逻辑决策和日志追踪。

### 六、渐进式子目录上下文加载（subdirectory_hints.py）

**文件**：`agent/subdirectory_hints.py`

启动时 `prompt_builder.py` 只加载工作目录（CWD）的上下文文件（AGENTS.md / CLAUDE.md / .cursorrules）。`SubdirectoryHintTracker` 解决的问题是：agent 深入子目录工作时，那些子目录的上下文规则文件无法被预先加载——因为在 agent 做出第一次工具调用之前，系统根本不知道它会进入哪个子目录。

#### 懒加载触发机制

`check_tool_call(tool_name, tool_args)` 在每次工具调用返回后被调用。它从工具参数中提取路径：
- **直接路径参数**：检查 `path`、`file_path`、`workdir` 等 key
- **shell 命令参数**（`terminal` 工具）：用 `shlex.split()` 解析命令字符串，提取包含 `/` 或 `.` 且不以 `-` 或 `http` 开头的 token

找到候选路径后，`_add_path_candidate()` 向上逐级扫描祖先目录（最多 `_MAX_ANCESTOR_WALK = 5` 级），直到遇到已加载过的目录为止。这确保读取 `project/src/main.py` 时能发现 `project/AGENTS.md`，即使 `project/src/` 自身没有任何提示文件。

`_loaded_dirs` set 记录所有已加载过的目录（初始化时预标记 CWD），防止同一目录的提示文件被重复注入。

#### 注入到工具结果而非 system prompt

发现新目录的上下文文件后，内容以追加形式拼接到工具调用结果字符串末尾（`tool_result += hints`），而不是修改 system prompt：

```
[Subdirectory context discovered: backend/src/utils/AGENTS.md]
<文件内容>
```

这个设计是本模块最值得关注的架构决策：**system prompt 保持不变，prefix cache 命中率不下降**。每次工具结果注入对模型来说是 KV cache 尾部的新 token，不影响 system prompt 部分的 cache key。与之对比，若将子目录上下文注入到 system prompt，则每发现一个新目录都会使整个 system prompt 缓存失效。

发现顺序遵循 `_HINT_FILENAMES` 列表（`AGENTS.md > agents.md > CLAUDE.md > claude.md > .cursorrules`），每个目录首次命中即停（"first-wins"，与启动加载策略一致）。每个文件内容上限 8,000 字符（`_MAX_HINT_CHARS`），超出部分截断并附加计数提示。

加载前调用 `_scan_context_content()`（与 `prompt_builder.py` 共享），执行与启动阶段相同的安全过滤。

### 七、辅助 LLM 路由

`call_llm(task="compression", ...)` 经过 `_resolve_task_provider_model()` 的四级优先级解析（显式参数 > 环境变量 `CONTEXT_COMPRESSION_MODEL` > config `auxiliary.compression.*` > config `compression.summary_model` 向后兼容）。超时读自 `auxiliary.compression.timeout`，默认 30 秒。失败进入 10 秒 jitter 重试退避（`jittered_backoff()`）。

## 关键代码路径

| 场景 | 文件 | 入口 |
|------|------|------|
| 对话时压缩核心算法 | `agent/context_compressor.py` | `ContextCompressor.compress()` |
| 工具输出剪枝 | `agent/context_compressor.py` | `_prune_old_tool_results()` |
| 结构化摘要生成 | `agent/context_compressor.py` | `_generate_summary()` |
| 工具对完整性修复 | `agent/context_compressor.py` | `_sanitize_tool_pairs()` |
| 压缩触发检测 | `run_agent.py` | `~L8858` `should_compress(_real_tokens)` |
| 压缩协调（记忆/会话/缓存） | `run_agent.py` | `_compress_context()` `~L5835` |
| 上下文长度解析 | `agent/model_metadata.py` | `get_model_context_length()` |
| token 粗估 | `agent/model_metadata.py` | `estimate_tokens_rough()` / `estimate_messages_tokens_rough()` / `estimate_request_tokens_rough()` |
| API 错误探测解析 | `agent/model_metadata.py` | `parse_context_limit_from_error()` + `get_next_probe_tier()` |
| 轨迹后压缩 | `trajectory_compressor.py` | `TrajectoryCompressor.compress_trajectory()` |
| 轨迹并发批处理 | `trajectory_compressor.py` | `process_entry_async()` |
| @-引用解析 | `agent/context_references.py` | `parse_context_references()` |
| @-引用展开（async） | `agent/context_references.py` | `preprocess_context_references_async()` |
| 敏感路径拦截 | `agent/context_references.py` | `_ensure_reference_path_allowed()` |
| 子目录上下文懒加载 | `agent/subdirectory_hints.py` | `SubdirectoryHintTracker.check_tool_call()` |
| 祖先目录扫描 | `agent/subdirectory_hints.py` | `_add_path_candidate()` |

## 设计亮点

### 1. 迭代摘要更新（Iterative Summary Update）

`_previous_summary` 存储上次压缩的摘要内容，下次压缩时作为前置上下文传给 LLM，生成"增量更新版"而非重新从零生成。这解决了多次压缩后信息累积丢失的问题：每次新摘要都继承前次摘要中的 Goal、Files、Decisions，只更新 Progress 状态。会话切换时 `reset_session_state()` 显式清除 `_previous_summary`，防止会话间污染（注释 `#2635`）。

### 2. Token-Budget 尾部保护

传统的"保护最后 N 条消息"策略无法适应不同大小的上下文窗口（同样是 20 条消息，在 8K 和 1M 窗口中意义完全不同）。Hermes 改为按 token 预算保护尾部（`tail_token_budget = threshold_tokens * target_ratio`），让保护范围随模型窗口自动缩放。

### 3. 工具对孤儿修复（_sanitize_tool_pairs）

压缩后极易产生两类 API 拒绝错误：tool result 引用不存在的 call_id，或 assistant 的 tool_calls 没有对应 result。`_sanitize_tool_pairs()` 在每次压缩后做一次全量扫描修复，保证消息列表在 API 层面始终合法，而不是把这个问题抛给调用方处理。

### 4. tools schema 纳入 token 估算

`estimate_request_tokens_rough()` 把工具 schema 字符串纳入 token 计算。50+ 工具的 Hermes 实例中，schema 序列化额外消耗 20-30K token，若只统计消息 token 会系统性低估实际 prompt 大小，导致压缩过晚触发。

### 5. 双阶段 token 探测与持久化

上下文探测不依赖预设表格，而是从 API 错误信息中实时解析真实限制（`parse_context_limit_from_error()`），并写入本地 YAML 缓存（key 为 `model@base_url`），后续会话零探测直接复用。探测结果按 `_context_probe_persistable` 标志区分"真实模型能力"和"订阅层限制"，后者不持久化（避免错误学习到临时性的低限值）。

### 6. @-引用的双重防御：工作区沙箱 + 凭证黑名单

`_resolve_path()` 强制所有相对路径在 `allowed_root`（默认 `cwd`）内解析，防止路径遍历（`@file:../../.env`）。`_ensure_reference_path_allowed()` 再叠加一层凭证文件/目录黑名单，覆盖 SSH 密钥、云凭证、Hermes 内部目录等。两道防线串联，不依赖用户的良性输入。

### 7. 子目录上下文注入到工具结果（prefix cache 保护）

将懒加载的子目录上下文（AGENTS.md 等）追加到工具调用结果而非 system prompt，是一个"以正确性换缓存效率"的精确取舍：system prompt 在整个会话期间保持字节级不变，所有 system prompt token 的 KV cache 始终命中；新发现的上下文以工具结果 suffix 的形式进入 KV cache 尾部，不影响已有 cache 前缀。对于高频、长会话场景，这一设计可显著降低推理成本。

### 8. 压缩前记忆持久化

`_compress_context()` 在调用核心算法前先调用 `flush_memories()` 和 `memory_manager.on_pre_compress()`，确保 agent 在"记忆消失前"有机会将重要信息写入外部记忆系统。压缩和记忆系统形成了协作关系，而非相互独立。

## 局限性

### 1. 摘要生成失败时静默丢失中间段

失败冷却期（600 秒）内，中间 turn 被直接删除，没有任何替代内容。与 Claude Code 的熔断（停止压缩但保留内容）相比，这种策略风险更大——在高频调用或辅助模型不稳定时，关键上下文可能无声无息地丢失。

### 2. 粗估误差问题

对话时压缩使用 4 chars/token 粗估，工具输出等结构化内容往往比自然语言 token 密度更高（如 JSON、代码），实际误差可能超过 30%。与后处理压缩使用真实 tokenizer 相比，这是一个有意识的工程权衡（避免对话时压缩引入 HuggingFace 依赖），但会导致阈值判断不精确。

### 3. 压缩触发点的 token 滞后

`should_compress()` 基于上一轮 API 响应返回的 `prompt_tokens`，此时不含本轮 tool result 的 token。若单条工具输出极大（如读取大文件），可能在下轮 API 调用时才触发压缩，而此时 prompt 已经超限（只能依赖 API 错误 → 探测 → 再压缩的降级路径）。

### 4. 轨迹压缩无迭代摘要

`TrajectoryCompressor` 每次独立生成摘要，不维护 `_previous_summary`。对于同一条长轨迹需要多次压缩的情况（如 RL 训练数据），信息的跨轮积累完全依赖单次摘要的覆盖范围，无法复用前次摘要的精炼结果。

### 5. 后处理压缩单点故障

`TrajectoryCompressor._init_tokenizer()` 失败（如 tokenizer 模型下载失败）会直接 `raise RuntimeError`，整个批处理任务中断。缺乏按轨迹粒度的隔离降级机制。

## 来源

- 源码版本：hermes-agent 0.16.0
- 分析文件：`agent/context_compressor.py`、`agent/model_metadata.py`、`trajectory_compressor.py`、`run_agent.py`（相关片段）、`agent/context_references.py`、`agent/subdirectory_hints.py`
- 分析深度：源码级
