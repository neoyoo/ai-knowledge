---
title: "finetuning-system——agentscope"
category: L2
parent: "[[finetuning-system]]"
source: "agentscope"
source_version: "0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12"
concept: "finetuning-system"
created: "2026-04-15"
updated: "2026-04-15"
confidence: high
---

## 概述

AgentScope 的 tuner/ 模块提供两条平行的优化路径：一是通过 Trinity-RFT 框架进行真实权重微调（支持 GRPO/SFT 等强化学习算法，默认 `multi_step_grpo`），二是通过 DSPy MIPROv2 进行无梯度 prompt 自动优化。此外还有一个独立的 `select_model` 功能，用于在多个候选模型上跑基准评测后自动选出最优模型。三条路径共享同一套 workflow/judge 函数契约，用户只需实现业务逻辑，训练/优化基础设施由模块托管。

---

## 架构分析

### 整体结构

```
tuner/
├── _tune.py            # 主入口：tune() → Trinity-RFT
├── _algorithm.py       # AlgorithmConfig（算法参数 Pydantic 模型）
├── _config.py          # _to_trinity_config() 转换层 + 函数签名校验
├── _workflow.py        # WorkflowType / WorkflowOutput 类型定义
├── _judge.py           # JudgeType / JudgeOutput 类型定义
├── _model.py           # TunerModelConfig / TinkerConfig（LoRA）
├── _dataset.py         # DatasetConfig（HuggingFace 兼容）
├── prompt_tune/
│   ├── _config.py      # PromptTuneConfig（DSPy 优化参数）
│   ├── _tune_prompt.py # tune_prompt() → DSPy MIPROv2
│   └── _wrapper.py     # _OptimizablePrompt + _WorkflowWrapperModule（DSPy 桥接）
└── model_selection/
    ├── _model_selection.py   # select_model()：并发评测 + 最优选择
    └── _built_in_judges.py   # avg_time_judge / avg_token_consumption_judge
```

### 三条优化路径

**路径 A：权重微调（tune()）**

用户提供 `workflow_func`（定义 agent 如何使用模型完成任务）和可选的 `judge_func`（外部评判奖励），调用 `tune()` 后全部委托给 Trinity-RFT 框架执行。AgentScope 在这里的作用是：将 Python 函数/Pydantic 配置对象转换为 Trinity-RFT 的 YAML 配置结构，并处理阿里云 PAI DLC 集群的 Ray 生命周期（`setup_ray_cluster` / `stop_ray_cluster`）。

**路径 B：Prompt 自动优化（tune_prompt()）**

基于 DSPy MIPROv2。核心技巧是将 `system_prompt` 字符串包装成 DSPy 的 `Predict` 子类 `_OptimizablePrompt`，令 MIPROv2 把它当作可优化的"指令字段"来修改，而实际执行仍走用户的 async workflow 函数。优化完成后抽取 `result.predictor.get_current_prompt()` 返回优化后的 prompt 字符串。

**路径 C：模型自动选择（select_model()）**

在多个 `ChatModelBase` 候选上用异步 Semaphore 控制并发度，对数据集每条样本跑 workflow → judge，累计平均 reward，选出最高分模型。同时通过 OpenTelemetry 收集每次 LLM 调用的 token 用量，注入到 judge 的 `response.metrics` 里，内置 judge 可据此按时间或 token 成本选模型。

### 数据流（路径 A 微调）

```
用户代码
  │
  ├── workflow_func(task, model, [auxiliary_models], [logger]) → WorkflowOutput
  │     └── WorkflowOutput.reward | .response | .metrics
  │
  └── judge_func(task, response, [auxiliary_models], [logger]) → JudgeOutput
        └── JudgeOutput.reward | .metrics
            ↓
tune(workflow_func, judge_func, train_dataset, model, algorithm, ...)
  │
  ├── check_workflow_function()  # 签名校验：必须含 task/model，禁止 *args/**kwargs
  ├── check_judge_function()     # 签名校验：必须含 task/response
  ├── _to_trinity_config()       # 转换为 trinity.common.config.Config
  │     ├── _load_config_from_path_or_default()  # 读文件或生成默认 YAML 模板
  │     ├── 注入 workflow_name = "agentscope_workflow_adapter_v1"
  │     ├── 映射 DatasetConfig → TasksetConfig
  │     ├── 映射 TunerModelConfig → InferenceModelConfig（+Tinker LoRA）
  │     └── 映射 AlgorithmConfig → config.algorithm / optimizer / buffer / trainer
  │
  └── trinity.cli.launcher.run_stage(config.check_and_update())
```

### 数据流（路径 B Prompt 优化）

```
tune_prompt(workflow, init_system_prompt, judge_func, train_dataset, ...)
  │
  ├── load HuggingFace dataset → dspy.Example 列表
  ├── _WorkflowWrapperModule(workflow, init_system_prompt)
  │     └── self.predictor = _OptimizablePrompt(init_prompt)
  │           └── 继承 dspy.Predict，将 system_prompt 暴露为 signature.instructions
  │
  ├── dspy.MIPROv2(metric=lambda: judge_func(data.inp, output), auto="light")
  ├── optimizer.compile(module, trainset=dspy_trainset)
  │     └── 每次 forward：
  │           predictor.sync_instruction()     # DSPy 修改后的指令同步到内部状态
  │           asyncio.run(workflow(task=inp, system_prompt=current_prompt))
  │           → WorkflowOutput.response 传给 metric
  │
  └── result.predictor.get_current_prompt() → 返回优化后 prompt 字符串
```

---

## 关键代码路径

### 1. tune() 主入口

```python
# tuner/_tune.py
def tune(
    *,
    workflow_func: WorkflowType,          # async (task, model, ...) -> WorkflowOutput
    judge_func: JudgeType | None,         # async (task, response, ...) -> JudgeOutput
    train_dataset: DatasetConfig | None,
    eval_dataset: DatasetConfig | None,
    model: TunerModelConfig | None,
    auxiliary_models: dict[str, TunerModelConfig] | None,
    algorithm: AlgorithmConfig | None,
    project_name: str | None,
    experiment_name: str | None,
    monitor_type: str | None,
    config_path: str | None,
) -> None
```

调用链：
```
tune()
  └── check_workflow_function(workflow_func)   # _config.py: 检查 essential_params=['task','model']
  └── _to_trinity_config(...)                  # _config.py: 构建 trinity.common.config.Config
        └── _load_config_from_path_or_default(config_path)
              # 无 config_path 时：写临时 YAML（默认 multi_step_grpo）→ trinity.load_config()
        └── config.buffer.explorer_input.taskset.workflow_args.update({"workflow_func": ..., "judge_func": ...})
        └── 各字段逐一 _set_if_not_none(config, field, value)
  └── trinity.cli.launcher.run_stage(config.check_and_update())
```

### 2. 函数签名校验

```python
# tuner/_config.py
def _check_function_signature(
    func: Callable,
    essential_params: List[str],
    optional_params: List[str] | None = None,
) -> None
```

- 遍历 `inspect.signature(func).parameters`，禁止 `*args` / `**kwargs`
- 所有参数必须属于 `essential_params | optional_params`，多余参数直接 raise
- workflow 允许参数：`task, model, auxiliary_models, logger`
- judge 允许参数：`task, response, auxiliary_models, logger`

### 3. TinkerConfig（运行时 LoRA）

```python
# tuner/_model.py
class TinkerConfig(BaseModel):
    rank: int = 16          # LoRA rank
    train_mlp: bool = True
    train_attn: bool = True
    train_unembed: bool = True   # 包括 unembedding 层，比标准 LoRA 更激进
    seed: int | None = None
    base_url: str | None = None  # 外部 Tinker 服务地址
```

映射路径：`TinkerConfig.get_config()` → `trinity.common.config.TinkerConfig` → `config.model.tinker.enable = True`

### 4. _to_trinity_config() 中的算法参数映射

```python
# tuner/_config.py
config.algorithm.algorithm_type = algorithm.algorithm_type   # e.g. "multi_step_grpo"
config.algorithm.repeat_times   = algorithm.group_size       # GRPO 中的组采样数
config.algorithm.optimizer.lr   = algorithm.learning_rate
config.buffer.batch_size        = algorithm.batch_size
config.trainer.save_interval    = algorithm.save_interval_steps
config.explorer.eval_interval   = algorithm.eval_interval_steps
```

注意 `group_size` 映射到 Trinity 的 `repeat_times`，即 GRPO 每个 prompt 生成的候选数。

### 5. tune_prompt() 核心桥接

```python
# tuner/prompt_tune/_wrapper.py
class _OptimizablePrompt(dspy.Predict):
    def __init__(self, init_prompt: str):
        super().__init__("input -> output")
        self._sys_prompt = init_prompt
        self.instructions = self._sys_prompt
        self.signature.instructions = self.instructions   # DSPy 优化目标

    def sync_instruction(self) -> None:
        self.instructions = self.signature.instructions   # 优化后同步回内部状态
        self._sys_prompt = self.instructions

    def get_current_prompt(self) -> str:
        return self._sys_prompt

class _WorkflowWrapperModule(dspy.Module):
    def forward(self, inp: Any) -> Any:
        self.predictor.sync_instruction()
        current_prompt = self.predictor.get_current_prompt()
        result = asyncio.run(self._workflow(task=inp, system_prompt=current_prompt))
        return result.response   # 传给 MIPROv2 的 metric 函数
```

### 6. select_model() 并发评测

```python
# tuner/model_selection/_model_selection.py
async def select_model(
    *,
    workflow_func: WorkflowType,
    judge_func: JudgeType,
    train_dataset: DatasetConfig,
    candidate_models: Sequence[ChatModelBase],
    max_threads: int = 2,
) -> Tuple[ChatModelBase, Dict[str, float]]
```

调用链：
```
select_model()
  └── _load_dataset(train_dataset)
  └── for model in candidate_models:
        semaphore = asyncio.Semaphore(max_threads)
        tasks = [evaluate_with_semaphore(idx, sample) for ...]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        └── _evaluate_single_sample(sample, model, workflow_func, judge_func, exporter)
              └── OpenTelemetry span: "Solution_{task_id}_{repeat_id}"
              └── workflow_output = await workflow_func(task=sample, model=model)
              └── 从 exporter.cnt[task_id][repeat_id]["chat_usage"] 累计 token 用量
              └── judge_output = await judge_func(task=sample, response={"response":..., "metrics":...})
        _process_evaluation_results(results, ...) → avg_reward
  └── 返回 best_model（avg_reward 最高者）
```

---

## 设计亮点

**1. Trinity-RFT 作为黑盒训练后端，AgentScope 只做配置转换**

`tune()` 函数本身几乎没有训练逻辑，全部委托给 Trinity-RFT 的 `run_stage()`。AgentScope 的贡献是将 Python 函数对象（`workflow_func`, `judge_func`）直接序列化进 Trinity 的 `workflow_args` 字典，通过 `"agentscope_workflow_adapter_v1"` 适配层在 Trinity 的 explorer 进程里被调用。这种设计使 AgentScope 能随 Trinity-RFT 演进而不需要自己维护分布式训练基础设施。

**2. 函数签名作为协议的严格校验**

workflow/judge 函数的接口通过 `_check_function_signature()` 在运行前强制校验，禁止 `*args`/`**kwargs`（因为 Trinity 侧需要按名称注入参数），且参数白名单固定（`task, model, auxiliary_models, logger`）。这是一种轻量级的"协议即代码"做法，比文档约定更可靠。

**3. DSPy prompt 优化中的 signature 劫持技巧**

`_OptimizablePrompt` 继承 `dspy.Predict` 但重写 `forward` 为 NotImplementedError，只用 DSPy 的 `signature.instructions` 字段来持久化 prompt 文本。MIPROv2 在优化循环里修改 `signature.instructions`，`sync_instruction()` 在每次 `forward` 前将其同步回 `_sys_prompt`。这样 DSPy 的 prompt 优化框架完全不需要知道 AgentScope workflow 的内部实现，只看到一个"指令可优化的 Module"。

**4. OpenTelemetry 用于 token 追踪**

`select_model` 中通过 OpenTelemetry baggage（`task_id`, `repeat_id`）标记每次 LLM 调用，用 `_InMemoryExporter` 收集 span 数据提取 `chat_usage`，将 token 用量注入 `workflow_output.metrics`。这使内置 judge（`avg_token_consumption_judge`）无需侵入 workflow 代码就能获得 token 成本数据。

**5. Tinker LoRA：推理时可更新权重**

`TinkerConfig` 对应 Trinity-RFT 中的 Tinker 机制，允许在推理引擎运行期间动态注入 LoRA 权重更新（`train_mlp + train_attn + train_unembed` 均可独立控制）。这比传统"训练→停止推理→加载新权重→重启推理"流程更高效，支持在线 GRPO 训练。

**6. 默认算法选 multi_step_grpo**

`AlgorithmConfig.algorithm_type` 默认值是 `"multi_step_grpo"` 而非 SFT，注释明确说"推荐用于大多数 agent tuning 场景"。这反映了 AgentScope 的工程判断：agent 任务（多步推理、工具调用）天然适合 GRPO 的组采样奖励设计，SFT 仅适合有监督标注数据的场景。

---

## 局限性

**1. 强依赖 Trinity-RFT，安装门槛高**

`tune()` 在运行时才 import `trinity`，任何无 Trinity-RFT 环境的调用都会抛 ImportError。Trinity-RFT 本身依赖 CUDA、vLLM、Ray，生产环境部署复杂度高。模块对 Trinity 的版本锁定也不透明（无显式版本约束）。

**2. 配置转换层是硬编码映射**

`_to_trinity_config()` 中字段映射是逐一硬编码的（`algorithm.group_size` → `config.algorithm.repeat_times` 等）。当 Trinity-RFT 内部字段名变更时，这层映射会静默失效——在调用 `config.check_and_update()` 之前不会报错。

**3. tune_prompt() 用 asyncio.run() 同步化 async workflow，不支持事件循环嵌套**

`_WorkflowWrapperModule.forward()` 内部调用 `asyncio.run()`，若用户代码本身已在 async 上下文中运行（如 Jupyter notebook、FastAPI handler），会触发 "This event loop is already running" 错误。

**4. model_selection 的 max_threads=2 默认值过低**

`select_model` 默认并发度为 2，对于大规模数据集评测效率较低。且 Semaphore 是按模型重建的（每个模型单独一个 Semaphore），无法跨候选模型全局控制并发。

**5. 无 SFT 直接支持**

`AlgorithmConfig.algorithm_type` 虽然文档中提到 `'sft'` 作为示例，但模块本身没有任何 SFT 特有的数据格式处理（如对话格式转换、chat template 应用）。SFT 数据准备完全依赖用户自行处理，且无 validation。

**6. prompt_tune 与 tune 的 workflow 函数签名不兼容**

- `tune()` 的 workflow 签名要求：`(task, model, [auxiliary_models], [logger])`
- `tune_prompt()` 的 workflow 签名要求：`(task, system_prompt, [...])`

两条路径的 workflow 函数不可复用，用户需要维护两套 workflow 实现才能同时使用权重微调和 prompt 优化。

**7. 云端 DLC 支持仅通过环境变量控制**

`USE_ALIYUN_PAI_DLC=1` 环境变量决定是否启用阿里云 PAI DLC 集群，这是阿里内部基础设施的耦合，对非阿里云用户完全透明（代码路径走 if 分支跳过），但也意味着对其他云平台的支持需要自行扩展。

---

## 来源

- 源码版本：`0ff492c3508e532d2a33234dfe4a299833ce866c 2026-04-12`
- 分析深度：源码级
- 关键文件：
  - `tuner/_tune.py` — 主入口
  - `tuner/_config.py` — Trinity 配置转换层
  - `tuner/_model.py` — TunerModelConfig + TinkerConfig
  - `tuner/_algorithm.py` — AlgorithmConfig
  - `tuner/_workflow.py` / `_judge.py` — 类型协议定义
  - `tuner/prompt_tune/_tune_prompt.py` — DSPy MIPROv2 集成
  - `tuner/prompt_tune/_wrapper.py` — DSPy 桥接层
  - `tuner/model_selection/_model_selection.py` — 自动模型选择
  - `tuner/model_selection/_built_in_judges.py` — 内置 judge 函数
