---
template: multi-agent-orchestrator
scenario: 多 Agent 任务编排
tags: [multi-agent, orchestration, delegation, task-routing]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: advanced
related_patterns: [prompt-chaining, react, structured-output, system-prompt-design]
---

# Multi-Agent Orchestrator 模板

> 把复杂任务交给一个 Orchestrator 分解、分派给多个 Worker Agent 并行执行，最后汇总结果。

## 场景描述

当一个任务太复杂、需要多种专业能力、或者可以并行拆解时，单个 Agent 处理的质量会下降。这时用 Orchestrator-Workers 模式：Orchestrator 负责理解目标、分解任务、分配给对应角色的 Worker，Workers 各自在自己的领域内独立完成子任务，最后 Orchestrator 汇总成最终答案。

典型场景：

- 市场调研报告（数据收集 + 竞品分析 + 趋势研判 + 撰写 → 四个专业 Worker）
- 代码 Review（安全审查 + 性能分析 + 可读性评审 → 三个专业 Worker）
- 复杂问题回答（检索 + 推理 + 核实 + 撰写 → 流水线 Worker）

## 模板

### Orchestrator System Prompt

```text
你是一个任务编排者（Orchestrator）。你的职责分两个阶段：

**阶段一：任务分解**
收到用户请求后，将其分解为若干独立子任务，每个子任务分配给一个专业角色。

分解原则：
- 每个子任务必须独立可完成，不依赖其他子任务的中间状态（除非必须串行）
- 子任务粒度适中：一个 Worker 一次 LLM 调用能完成
- 不同类型的子任务分配给不同角色（researcher / analyst / writer / reviewer / coder 等）
- 如果子任务之间有顺序依赖，在 depends_on 字段中列明

分解阶段的输出格式（严格 JSON，不要附加解释文字）：
{
  "goal": "{{original_user_request}}",
  "tasks": [
    {
      "id": "t1",
      "role": "researcher",
      "instruction": "...",
      "output_format": "bullet list / json / prose",
      "depends_on": []
    },
    {
      "id": "t2",
      "role": "analyst",
      "instruction": "...",
      "output_format": "json",
      "depends_on": ["t1"]
    }
  ]
}

**阶段二：结果汇总**
收到所有子任务的结果后，综合整理为最终答案，要求：
- 结构清晰，有层次感
- 不重复罗列原始 Worker 输出，而是提炼整合
- 指出关键结论和行动建议
- 如果某个 Worker 返回了失败标记（[FAILED]），在汇总中说明该部分缺失并给出替代建议
```

---

### Worker System Prompt 模板

```text
你是一个专业的{{role}}。你只负责完成被分配给你的单一子任务，严格不超出范围。

**你的子任务：**
{{instruction}}

**输出格式要求：**
{{output_format}}

{{#if upstream_context}}
**上游任务结果（参考信息，不得直接复制粘贴）：**
{{upstream_context}}
{{/if}}

**约束：**
- 直接输出结果，不要解释你在做什么，不要重复任务描述
- 如果任务中有信息缺失导致你无法完成，输出：[BLOCKED] 原因：<说明缺失什么>
- 如果任务超出你的能力范围，输出：[FAILED] 原因：<说明为什么>
- 不要尝试完成本任务以外的工作
```

---

### Task Envelope 格式（传递给 Worker 的标准化结构）

```json
{
  "task_id": "{{task_id}}",
  "role": "{{role}}",
  "instruction": "{{instruction}}",
  "output_format": "{{output_format}}",
  "upstream_context": "{{upstream_results_if_any}}",
  "constraints": {
    "max_output_tokens": 1024,
    "must_use_upstream": true
  }
}
```

---

### 结果汇总 Prompt

```text
以下是对用户请求「{{original_request}}」拆解后各子任务的执行结果：

{{#each task_results}}
---
[子任务 {{id}} · {{role}}]
{{result}}
{{/each}}

请综合以上结果，为用户提供一份完整、结构清晰的最终答案：
- 将各部分结果有机整合，不要简单罗列
- 突出关键结论和可执行的建议
- 如有子任务失败（[FAILED] 标记），说明该部分缺失，并基于现有信息给出最佳估计
```

---

### Python 实现骨架

```python
import json
import asyncio
import anthropic

client = anthropic.Anthropic()

ORCHESTRATOR_SYSTEM = "..."  # 上方 Orchestrator System Prompt
WORKER_SYSTEM_TMPL = "..."   # 上方 Worker System Prompt 模板

def run_orchestrator(user_request: str) -> str:
    # Phase 1: 分解任务
    decompose_resp = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=1024,
        system=ORCHESTRATOR_SYSTEM,
        messages=[{"role": "user", "content": f"请分解以下任务：{user_request}"}]
    )
    plan = json.loads(decompose_resp.content[0].text)
    tasks = plan["tasks"]

    # Phase 2: 按依赖顺序执行 Workers
    results: dict[str, str] = {}
    for task in tasks:
        upstream = "\n\n".join(
            f"[{dep}]\n{results[dep]}" for dep in task["depends_on"] if dep in results
        )
        result = execute_worker(task, upstream_context=upstream)
        results[task["id"]] = result

    # Phase 3: Orchestrator 汇总
    summary_input = "\n\n".join(
        f"[子任务 {tid} · {t['role']}]\n{results[tid]}"
        for t in tasks
        for tid in [t["id"]]
    )
    final_resp = client.messages.create(
        model="claude-opus-4-5",
        max_tokens=2048,
        system=ORCHESTRATOR_SYSTEM,
        messages=[{
            "role": "user",
            "content": (
                f"原始任务：{user_request}\n\n"
                f"各子任务结果：\n{summary_input}\n\n"
                "请综合汇总最终答案。"
            )
        }]
    )
    return final_resp.content[0].text


def execute_worker(task: dict, upstream_context: str, max_retries: int = 2) -> str:
    """带重试和错误标记的 Worker 执行"""
    system_prompt = WORKER_SYSTEM_TMPL.format(
        role=task["role"],
        instruction=task["instruction"],
        output_format=task.get("output_format", "prose"),
        upstream_context=upstream_context,
    )
    for attempt in range(max_retries + 1):
        resp = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=task.get("constraints", {}).get("max_output_tokens", 1024),
            system=system_prompt,
            messages=[{"role": "user", "content": "请完成你的子任务。"}]
        )
        result = resp.content[0].text.strip()

        # 非失败状态直接返回
        if not result.startswith("[FAILED]") and not result.startswith("[BLOCKED]"):
            return result

        # 还有重试机会则继续
        if attempt < max_retries:
            system_prompt += f"\n\n[系统提示] 上次未能完成任务，请再试一次，确保输出符合格式要求。"

    # 所有重试失败
    return f"[FAILED] 任务 {task['id']}（{task['role']}）在 {max_retries + 1} 次尝试后仍未完成"
```

---

### 错误处理：Worker 失败时的策略

```python
def handle_failed_tasks(results: dict[str, str], tasks: list[dict]) -> dict[str, str]:
    """
    针对失败 Worker 的三种降级策略：
    1. BLOCKED：缺少上游信息 → 尝试提供默认上下文后重试
    2. FAILED：能力超限 → 简化任务描述后重试，或标记为缺失
    3. 超时 / 异常 → 直接标记为缺失，汇总时说明
    """
    cleaned = {}
    for task in tasks:
        tid = task["id"]
        result = results.get(tid, "[FAILED] 未执行")

        if result.startswith("[BLOCKED]"):
            # 提供宽松上下文，降级重试
            fallback = execute_worker(
                {**task, "instruction": f"（简化版）{task['instruction']}"},
                upstream_context="（上游信息不可用，请基于常识完成）"
            )
            cleaned[tid] = fallback
        elif result.startswith("[FAILED]"):
            # 直接标记，汇总阶段处理
            cleaned[tid] = result
        else:
            cleaned[tid] = result

    return cleaned
```

## 自定义指南

**替换要点：**

| 占位符 | 说明 | 示例 |
|--------|------|------|
| `{{role}}` | Worker 的专业角色名称 | `researcher`、`security_reviewer`、`data_analyst` |
| `{{instruction}}` | 具体子任务描述，越精确越好 | "从以下文本中提取所有公司名称，输出 JSON 数组" |
| `{{output_format}}` | 对输出格式的约束 | `json`、`bullet list`、`prose（不超过 300 字）` |
| `{{upstream_context}}` | 上游任务的结果，按需注入 | 前置任务的原始输出文本 |
| `{{original_request}}` | 用户原始请求，用于汇总阶段 | 保持原文，不要改写 |

**角色库建议（直接复用）：**

- `researcher`：信息检索与整理，输出 bullet list
- `analyst`：数据分析与模式识别，输出 JSON 或结构化表格
- `writer`：将结构化信息写成自然语言，输出 prose
- `reviewer`：审查内容的准确性和完整性，输出问题列表
- `coder`：代码生成与调试，输出代码块
- `critic`：批判性审查，找漏洞和风险，输出风险列表

## 使用示例

**场景：技术方案评审**

```python
request = "评审这份微服务架构方案：[架构文档]"

# Orchestrator 会自动分解为：
# t1: security_reviewer → 安全风险扫描
# t2: performance_analyst → 性能瓶颈分析（depends_on: []，可与 t1 并行）
# t3: architecture_critic → 架构合理性评审（depends_on: []）
# t4: writer → 综合报告撰写（depends_on: [t1, t2, t3]）

result = run_orchestrator(request)
```

**场景：竞品分析报告**

```python
request = "分析 Notion、Obsidian、Roam Research 三款产品，给出产品策略建议"

# 自动分解为：
# t1: researcher → 收集三款产品的核心功能特性
# t2: researcher → 收集三款产品的用户口碑和痛点
# t3: analyst → 横向对比分析（depends_on: [t1, t2]）
# t4: writer → 撰写策略建议（depends_on: [t3]）
```

## 适配建议

**并行执行（加速）**

默认实现是串行执行（按依赖顺序）。对没有 `depends_on` 的任务，可以改用 `asyncio.gather` 并行发起，显著降低总延迟。

```python
async def run_parallel_workers(independent_tasks: list[dict]) -> dict[str, str]:
    async with asyncio.TaskGroup() as tg:
        futures = {
            task["id"]: tg.create_task(
                asyncio.to_thread(execute_worker, task, "")
            )
            for task in independent_tasks
        }
    return {tid: fut.result() for tid, fut in futures.items()}
```

**不同 Worker 用不同模型**

高强度推理任务（分析、评审）用 Opus，简单整理任务（格式化、提取）用 Haiku，降低成本。

```python
MODEL_BY_ROLE = {
    "researcher": "claude-haiku-4-5",
    "analyst": "claude-opus-4-5",
    "writer": "claude-sonnet-4-5",
    "reviewer": "claude-opus-4-5",
}
```

**任务上限**

单次编排建议不超过 8 个子任务。超过时，考虑分层编排（Orchestrator → Sub-Orchestrator → Workers）或缩小问题范围。

## 关联

关联模板：[[prompt-chaining]] · [[conversation-agent]]

关联 wiki：[[wiki/multi-agent]] · [[wiki/query-loop]] · [[wiki/tool-system]]

## 来源

- **Anthropic Claude Docs** — Multi-agent systems 设计指南
  [https://docs.anthropic.com/claude/docs/multi-agent-systems](https://docs.anthropic.com/claude/docs/multi-agent-systems)

- **DeerFlow** — Orchestrator-Workers 模式工程实现参考
  `raw/deer-flow`

- **neoagent v3.2b** — Task Envelope、Worker 角色授权、超时处理的生产级实现
  `raw/neoagent`
