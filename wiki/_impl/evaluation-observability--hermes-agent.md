---
title: "Evaluation & Observability — Hermes Agent"
category: L2
parent: "[[evaluation-observability]]"
source: hermes-agent
source_version: "0.16.0"
confidence: high
created: 2026-04-08
updated: 2026-06-10
---

## 概述

Hermes Agent 把 Evaluation & Observability 拆成两条相对独立的线：**运行时可观测性**（session 级别的 token/cost/tool 追踪 + SQLite 持久化）和 **RL 训练评估**（batch trajectory 生成 + WandB 指标 + 推理测试）。前者服务于日常 agent 运营（成本控制、使用分析），后者服务于模型训练闭环（数据生成 → 压缩 → 训练 → 指标回收）。设计理念是"本地优先"——没有外部 telemetry 平台依赖，所有数据持久化在 `~/.hermes/` 下的 SQLite 文件和日志中。

## 架构分析

### 一、Session 级可观测性：SessionDB

`hermes_state.py` 中的 `SessionDB` 是所有运行时 observability 的枢纽。SQLite + WAL 模式，`SCHEMA_VERSION = 6`（历经六次迭代演进），同时支持 CLI、Gateway（Telegram/Discord/Slack 等）、ACP Adapter 多条路径写入。

**sessions 表核心字段：**

| 字段 | 说明 |
|------|------|
| `source` | 来源平台（`'cli'`, `'telegram'`, `'discord'` 等），支持按平台过滤 |
| `message_count` / `tool_call_count` | 会话消息数 / 工具调用数（session 级聚合） |
| `input_tokens` / `output_tokens` | LLM 输入/输出 token 数（增量累加） |
| `cache_read_tokens` / `cache_write_tokens` | Prompt cache 读写 token（分别计）|
| `reasoning_tokens` | 推理 token（OpenAI o 系列、Kimi 等） |
| `billing_provider` | 计费来源提供商（`anthropic` / `openai` / `openrouter` 等） |
| `billing_base_url` | 实际请求 base URL（用于自托管端点识别） |
| `billing_mode` | 计费模式（`official_docs_snapshot` / `official_models_api` / `unknown` 等） |
| `estimated_cost_usd` | 估算费用（USD） |
| `actual_cost_usd` | 实际费用（USD，来自提供商返回值或外部账单） |
| `cost_status` | 费用状态（`actual` / `estimated` / `included` / `unknown`） |
| `cost_source` | 价格数据来源（`provider_cost_api` / `official_docs_snapshot` 等） |
| `pricing_version` | 价格快照版本字符串（如 `anthropic-prompt-caching-2026-03-16`） |
| `parent_session_id` | 压缩分裂后的 session 链（形成 lineage） |

**messages 表：** 存储完整消息历史，含 `reasoning`、`reasoning_details`、`codex_reasoning_items` 三个推理字段（v6 新增），以及 FTS5 虚拟表支持全文搜索。

**写竞争优化：** 多进程（Gateway + CLI + worktree agents）并发写一个 `state.db`。`SessionDB._execute_write()` 用 `BEGIN IMMEDIATE` + 随机 jitter 重试（20-150ms，最多 15 次），打破 convoy 效应；每 50 次写触发 `PRAGMA wal_checkpoint(PASSIVE)` 防止 WAL 文件无限增长。

### 二、费用计算管道：usage_pricing.py

`agent/usage_pricing.py` 实现了完整的费用计算框架，是 Hermes 最有设计感的模块之一。

**核心数据结构：**
- `CanonicalUsage`（frozen dataclass）：统一 token 格式，屏蔽 Anthropic / OpenAI / Codex 三种 API 返回差异
- `BillingRoute`：解析提供商路由（provider + model + base_url → billing_mode）
- `PricingEntry`（frozen dataclass）：存储每百万 token 价格，含 `pricing_version` 和 `fetched_at` 时间戳
- `CostResult`：最终费用结果，含 `status` 和 `source` 完整溯源

**定价解析优先级链（`get_pricing_entry()`）：**
1. `subscription_included` 路由（如 openai-codex）→ 直接返回零费用
2. OpenRouter → 调用 `/models` API 拉取实时价格（`_openrouter_pricing_entry()`）
3. 自定义 base_url → 调用目标端点 `/models` API（兼容 OpenAI spec）
4. 官方文档快照（`_OFFICIAL_DOCS_PRICING`）→ 覆盖 Anthropic、OpenAI、DeepSeek、Google Gemini 等主要模型

`normalize_usage()` 处理三种 API 格式差异：
- Anthropic：`cache_read_input_tokens` / `cache_creation_input_tokens` 直接分离
- OpenAI Chat：`prompt_tokens_details.cached_tokens` 需要从总量中减去净输入
- OpenAI Codex Responses：同 OpenAI Chat，字段名称不同

`resolve_billing_route()` 根据 model 字符串中的 `/` 分隔符推断提供商（如 `anthropic/claude-opus-4-20250514` → `anthropic`），支持 OpenRouter、自托管、本地推理等多种场景。

### 三、InsightsEngine：session 级分析仪表盘

`agent/insights.py` 中的 `InsightsEngine` 基于 `SessionDB` 的历史数据，生成多维度使用分析报告（对标 Claude Code 的 `/insights` 命令）。

关键分析维度：
- **overview**：total sessions/messages/tool_calls/tokens、estimated vs actual cost、session duration 统计
- **models**：按模型分桶的 token 消耗 + 费用分布
- **platforms**：按 source 字段（cli/telegram/discord 等）的平台维度拆分
- **tools**：工具调用频率排行（双源合并：`tool_name` 列 + `tool_calls` JSON 解析）
- **activity**：按小时/星期的活动模式分析

工具频率统计存在一个设计细节：CLI 路径写入的 `tool` role 消息中 `tool_name` 为 NULL，而 Gateway 路径会填写 `tool_name`。`InsightsEngine._get_tool_usage()` 同时从两个数据源提取（`tool_name` 列 + `assistant` 消息的 `tool_calls` JSON），取 max 去重，避免重复计数。

### 四、Batch Trajectory 生成：BatchRunner

`batch_runner.py` 是 Hermes 的 RL 数据生成引擎，两层并行架构：
- **外层**：`multiprocessing.Pool`（`num_workers` 个进程），处理多个 batch
- **内层**：每个 batch worker 串行处理本 batch 内的 prompts（避免 API rate limit 冲突）

**每条 trajectory 的 metadata 结构（`trajectory_entry`）：**
```python
{
    "prompt_index": int,
    "conversations": [...],        # OpenAI 格式消息历史
    "metadata": {"batch_num", "timestamp", "model"},
    "completed": bool,             # 是否自然完成
    "partial": bool,               # 是否因无效工具调用中途停止
    "api_calls": int,
    "toolsets_used": [...],
    "tool_stats": {tool: {count, success, failure}},   # 归一化：包含所有可能工具
    "tool_error_counts": {tool: failure_count},
}
```

**关键数据质量门禁：**

1. **零推理过滤**：`_extract_reasoning_stats()` 检查每个 assistant turn 是否含 `<REASONING_SCRATCHPAD>` 或 native `reasoning` 字段，若整个 trajectory 无任何推理输出则丢弃（`discarded_no_reasoning` 计数）
2. **工具名幻觉过滤**：合并阶段对比 `VALID_TOOLS`（来自 `TOOL_TO_TOOLSET_MAP`），含非法工具名的条目自动剔除
3. **Schema 归一化**：`_normalize_tool_stats()` / `_normalize_tool_error_counts()` 保证所有条目包含完整工具列表（缺失工具补零），避免 HuggingFace datasets 加载时 schema 不一致报错

**断点续跑：** `BatchRunner._scan_completed_prompts_by_content()` 扫描已有 batch 文件，以 prompt 文本内容为 key 匹配（而非 index），对 index 偏移和 dataset 重组有容错性。`atomic_json_write()` 保证 checkpoint 文件的原子写入；每批完成后立即更新 checkpoint，降低崩溃时的数据丢失范围。

### 五、Trajectory 压缩：TrajectoryCompressor

`trajectory_compressor.py` 对 RL 训练数据做 token budget 内压缩，含完整的 metrics tracking。

**压缩策略（中间删除法）：**
1. 保护头部：system、first human、first gpt、first tool（4 类 head turns）
2. 保护尾部：last N turns（默认 4）
3. 仅压缩中间区域：调用外部 LLM（默认 `google/gemini-3-flash-preview` via OpenRouter）生成摘要，替换为单条 human summary 消息
4. 目标 token 数：15250（`CompressionConfig.target_max_tokens`），摘要预算 750 tokens

**`TrajectoryMetrics` + `AggregateMetrics` 双层指标体系：**

| 指标 | 说明 |
|------|------|
| `original_tokens` / `compressed_tokens` | 压缩前后 token 数 |
| `compression_ratio` | 压缩比（保留至小数点后 4 位） |
| `turns_compressed_start_idx` / `turns_compressed_end_idx` | 压缩区域的起止 turn 索引 |
| `was_compressed` | 是否触发压缩 |
| `still_over_limit` | 压缩后仍超限（质量问题标记） |
| `summarization_api_calls` / `summarization_errors` | 摘要 LLM 调用次数 / 失败次数 |

聚合指标（`AggregateMetrics`）输出到 `compression_metrics.json`：平均压缩比、摘要 API 成功率、处理时长等，用于评估数据集质量。

`CompressionConfig` 支持从 YAML 加载，所有参数均可覆写；tokenizer 使用 HuggingFace `AutoTokenizer`（默认 `moonshotai/Kimi-K2-Thinking`），与目标训练模型对齐。

### 六、RL 训练工具链：rl_training_tool.py

`tools/rl_training_tool.py` 将 RL 训练生命周期封装为 agent 可调用的工具，核心工具链：

| 工具 | 功能 |
|------|------|
| `rl_list_environments()` | AST 扫描 `tinker-atropos/tinker_atropos/environments/`，发现 `BaseEnv` 子类 |
| `rl_select_environment(name)` | 动态 import 环境，通过 `config_init()` 或 `BaseEnvConfig` 反射提取可配置字段 |
| `rl_get_current_config()` | 返回 configurable_fields + locked_fields 分类列表 |
| `rl_edit_config(field, value)` | 修改可配置字段（locked 字段拒绝修改） |
| `rl_start_training()` | 生成 YAML config + 启动三进程组（run-api + launch_training.py + env.py serve） |
| `rl_check_status(run_id)` | 查询进程状态 + WandB 指标（rate-limited：30 分钟间隔） |
| `rl_get_results(run_id)` | 获取最终 WandB 指标 + history（10 samples） |
| `rl_test_inference()` | 在 3 个不同规模模型上跑 3 步推理测试（3×16=48 rollouts/model） |

**LOCKED_FIELDS 机制：** 基础设施参数（tokenizer、server URL、LoRA rank、learning rate 等）在代码中硬编码，`rl_edit_config()` 会拒绝对这些字段的修改。训练时从 `LOCKED_FIELDS` 深拷贝作为基础 config，再叠加用户可配置字段，生成 `configs/run_{run_id}.yaml`，确保训练的可复现性。

**WandB 指标监控（`rl_check_status()`）：**
```python
result["metrics"] = {
    "step": wandb_run.summary.get("_step", 0),
    "reward_mean": wandb_run.summary.get("train/reward_mean"),
    "percent_correct": wandb_run.summary.get("train/percent_correct"),
    "eval_percent_correct": wandb_run.summary.get("eval/percent_correct"),
}
```

**rl_test_inference() 的三模型测试策略：** 分别用 small（Qwen3-8B）/ medium（GLM-4.7 Flash）/ large（MiniMax M2.7）三个规模模型测试，检验环境在不同智力水平下的鲁棒性（防止只对强模型有效）。测试结果写入 `~/.hermes/logs/rl_training/inference_tests/`，同时上传 WandB 追踪。

### 七、HermesAgentLoop：RL 环境中的评估引擎

`environments/agent_loop.py` 中的 `HermesAgentLoop` 是 RL 环境（Atropos）内复用 hermes-agent 工具调用能力的适配层。

**`AgentResult` 数据结构：**
```python
@dataclass
class AgentResult:
    messages: List[Dict[str, Any]]     # 完整对话历史
    managed_state: Optional[Dict]      # ManagedServer 状态（Phase 2）
    turns_used: int                    # LLM 调用次数
    finished_naturally: bool           # 是否自然结束（vs 超 max_turns）
    reasoning_per_turn: List[Optional[str]]  # 逐 turn 推理内容
    tool_errors: List[ToolError]       # 工具执行错误记录
```

`_extract_reasoning_from_message()` 支持三种提供商格式的推理字段提取（`reasoning_content` / `reasoning` / `reasoning_details[].text`），并将提取的推理内容通过 `msg_dict["reasoning_content"]` 保留在消息历史中，供 Kimi-K2 等模型的 chat template 正确渲染。

工具调用错误通过 `ToolError` dataclass 结构化记录（turn、tool_name、arguments、error、tool_result），为 reward function 提供工具失败诊断能力。

**线程池设计：** `_tool_executor = ThreadPoolExecutor(max_workers=128)`，处理内部使用 `asyncio.run()` 的同步工具（Modal / Docker / Daytona 终端后端），避免与 Atropos 事件循环死锁。`resize_tool_pool()` 允许 `HermesAgentBaseEnv.__init__` 在运行时按 eval 并发量动态调整池大小。

### 八、日志体系：hermes_logging.py

`hermes_logging.py` 提供统一的日志初始化入口 `setup_logging()`：
- 双文件：`~/.hermes/logs/agent.log`（INFO+）和 `errors.log`（WARNING+），均用 `RotatingFileHandler`
- `RedactingFormatter` 过滤敏感信息（API key 等），确保 secret 不落盘
- 静默第三方噪音 logger（openai、httpx、asyncio、modal、websockets 等）
- 幂等设计：多次调用安全（`_logging_initialized` 哨兵变量）
- `mode` 参数区分上下文（`'cli'` / `'gateway'` / `'cron'`），Gateway 模式日志含 PID

RL 训练产生的三路进程日志（api、trainer、env）独立写入 `~/.hermes/logs/rl_training/api_{run_id}.log` 等，进程停止后由 `_stop_training_run()` 关闭文件句柄。

## 关键代码路径

- `hermes_state.py` — `SessionDB`，WAL 模式 SQLite，含 schema v1-v6 迁移，cost/token 全量追踪
- `agent/usage_pricing.py` — `normalize_usage()` 统一 token 格式，`estimate_usage_cost()` 多路径定价，`_OFFICIAL_DOCS_PRICING` 快照表
- `agent/insights.py` — `InsightsEngine.generate()`，多维度 session 分析
- `batch_runner.py` — `BatchRunner.run()`，multiprocessing Pool，trajectory 生成 + 断点续跑 + 数据质量过滤
- `trajectory_compressor.py` — `TrajectoryCompressor`，中间摘要压缩，`TrajectoryMetrics` + `AggregateMetrics`
- `tools/rl_training_tool.py` — RL 工具链，`LOCKED_FIELDS`，三进程 spawn，WandB 集成，rate-limited status check
- `environments/agent_loop.py` — `HermesAgentLoop.run()`，多格式推理提取，`ToolError` 结构化记录，128-线程工具池
- `environments/hermes_base_env.py` — `HermesAgentBaseEnv`，Atropos 集成基类，`ToolContext` 构建
- `hermes_logging.py` — `setup_logging()`，双文件 rotating log，RedactingFormatter

## 设计亮点

**1. 双模 token 计数（绝对值 + 增量）**

`SessionDB.update_token_counts()` 的 `absolute` 参数解决了 CLI vs Gateway 路径的不同累积方式。CLI 路径每次 API 调用后以增量（delta）更新；Gateway 路径缓存 agent 持有累积总量，以绝对值覆写。同一个函数两种语义，通过一个 bool 参数分离。

**2. pricing_version 快照可溯源**

每条 session 记录 `pricing_version`（如 `anthropic-prompt-caching-2026-03-16`），结合 `cost_source` 和 `cost_status`，任何历史费用数据都能追溯其定价来源和置信度。这是 cost audit 的基础设施，Claude Code 仅保存 `totalCostUSD` 而缺乏来源追踪。

**3. LOCKED_FIELDS 训练可复现性保障**

RL 训练的基础设施参数（tokenizer、server URL、LoRA rank、学习率、checkpoint 间隔等）硬编码在代码中，不暴露给 agent 修改。模型只能调整业务参数（数据集、训练步数、WandB 名称等）。这防止了 agent 为了"让训练跑起来"而乱改基础参数破坏实验条件。

**4. 三模型鲁棒性测试（rl_test_inference）**

在不同智力规模模型（8B / Flash / M2.7）上各跑 48 个 rollout，验证环境的鲁棒性。单一强模型测试通过不等于环境可用——小模型的输出格式更混乱，能暴露 prompt 构造或 verifier 的边界问题。

**5. 推理覆盖率作为数据质量门禁**

`_extract_reasoning_stats()` 检查 trajectory 中每个 assistant turn 是否有推理过程（scratchpad 或 native thinking tokens），整条 trajectory 无推理则丢弃。这保证 RL 训练数据包含推理链，避免用"只会输出答案"的样本训练推理模型。

**6. 内容匹配断点续跑（vs 索引匹配）**

`_scan_completed_prompts_by_content()` 以 prompt 文本内容为 key 做去重，而不是 index。当数据集被重排、截断或重组时，这种方式能正确识别已完成的工作，避免重复生成同一 prompt 的 trajectory。

## 局限性

**1. 无实时 streaming metrics**

Session 级 token/cost 数据只在 API 调用返回后更新，不支持实时查看当前运行中的消耗。长时间 batch run 中途无法精确知道当前费用。

**2. WandB 强依赖 + 无 fallback**

`rl_check_status()` 和 `rl_get_results()` 的指标获取都通过 WandB Python SDK 查询（`wandb.Api()`），若 WandB 不可用则整段指标为空，返回 `wandb_error` 字段。没有本地指标缓存或替代后端。

**3. 进程崩溃后的训练状态不可恢复**

`_active_runs` 是内存中的全局 dict，进程重启后所有 `RunState` 丢失。已经启动的三进程训练组（run-api + trainer + env）会成为孤儿进程，既无法通过 `rl_check_status()` 查询，也无法通过 `rl_stop_training()` 停止，需要手动 kill。

**4. 日志只有文件落地，无结构化 export**

`hermes_logging.py` 的输出只有两个 rotating 文本文件，没有结构化日志（JSON Lines）或外部 sink（Datadog、Sentry、OpenTelemetry）。大规模部署时 log 分析全靠 grep，没有 Claude Code 那种 Datadog 实时告警能力。

**5. rl_check_status 的 rate limit 过于粗放**

30 分钟全局 rate limit 是一刀切设计，无法区分"刚启动时的频繁检查"和"长时间运行后的例行监控"。在训练初期（前 30 分钟）无法获得任何状态反馈。

**6. Trajectory 压缩中间摘要的信息损失无法量化**

`TrajectoryCompressor` 记录了 token 压缩比，但没有追踪摘要质量（如关键工具调用是否被遗漏）。`still_over_limit` 标记了技术上的失败，但无法评估"压缩成功但摘要质量差"的情况。

## 来源

- 源码版本：Hermes Agent 0.16.0
- 分析深度：源码级
- 主要分析文件：`hermes_state.py`、`agent/usage_pricing.py`、`agent/insights.py`、`batch_runner.py`、`trajectory_compressor.py`、`tools/rl_training_tool.py`、`environments/agent_loop.py`、`environments/hermes_base_env.py`、`hermes_logging.py`、`rl_cli.py`
