---
pattern: tool-error-handling
category: tool-design
tags: [error, retry, fallback, degradation, resilience]
sources: [claude-code, deer-flow, openharness]
related_wiki: [wiki/tool-system, wiki/query-loop]
related_patterns: [tool-design-principles, permission-model]
---

# Tool Error Handling

> 工具调用失败时的处理策略——重试、降级、用户确认，确保 agent 不因一个工具失败而卡死。

## 本质

Agent 在真实环境中运行，真实环境会出错：文件不存在、API 超时、权限被拒绝、工具参数写错。没有容错设计的 agent，第一次碰到错误就卡死；有容错设计的 agent，能绕过障碍继续完成任务。

工具错误处理的核心问题是：**这个错误该怎么响应？** 不是所有错误都应该重试，不是所有错误都需要问用户，乱重试有时会造成更大的破坏（比如重复提交）。

## 错误类型分类

先分类，再决定响应策略。

| 错误类型 | 特征 | 响应策略 |
|---------|------|---------|
| **参数错误** | 工具返回参数格式不对、必填字段缺失、类型不匹配 | 不重试相同参数；模型需要修正参数后重试 |
| **暂时性失败** | 网络超时、服务暂时不可用（5xx）、资源被临时锁定 | 重试（指数退避）；超过次数后降级或报错 |
| **权限被拒** | 403、文件权限不足、操作超出沙箱范围 | 不重试；请求用户确认或提升权限 |
| **工具不存在** | 模型调用了注册表里没有的工具名 | 不重试相同名字；搜索相似功能的替代工具 |
| **资源不存在** | 文件不存在、URL 404、数据库记录不存在 | 视任务而定：有时需要先创建，有时是参数错误（路径拼写）|
| **输出溢出** | 工具返回内容超出 context 预算 | 切换到分段读取模式；或持久化到文件后返回路径 |

**关键原则：错误类型决定策略，不要无脑重试。**

参数错误重试相同参数 = 一直失败；权限错误自动重试 = 浪费请求、可能触发限流。

## is_error 标记

工具返回错误时，**必须使用 `is_error: true` 标记**，而不是把错误信息混入正常返回内容。

Anthropic tool_use API 的 `tool_result` 支持此字段：

```python
# 工具执行失败时的返回
tool_results.append({
    "type": "tool_result",
    "tool_use_id": tool_use_id,
    "content": "文件 /tmp/output.txt 不存在。请先用 write_file 工具创建该文件，或检查路径是否正确。",
    "is_error": True   # 关键：告诉模型这是错误，不是正常内容
})
```

没有 `is_error` 时，模型会把错误信息当作正常工具输出来处理，可能从"Error: file not found"里提取内容，产生荒谬的后续行为。加上 `is_error: true` 后，模型切换到错误处理推理模式，会考虑修复策略。

## 重试策略

### 简单重试（适合网络抖动）

```python
import time

def execute_with_retry(tool_fn, args, max_retries=3):
    for attempt in range(max_retries):
        try:
            result = tool_fn(args)
            return result
        except TemporaryError as e:
            if attempt == max_retries - 1:
                raise
            time.sleep(1)  # 简单等 1 秒
    raise Exception("max retries exceeded")
```

适合：偶发网络抖动、短暂资源锁定。不适合：参数错误、权限错误。

### 指数退避（适合服务限流/过载）

```python
def execute_with_backoff(tool_fn, args, max_retries=4, base_delay=1.0):
    for attempt in range(max_retries):
        try:
            return tool_fn(args)
        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt)  # 1s, 2s, 4s, 8s
            time.sleep(delay)
```

适合：API 限流（429）、服务过载（503）。指数退避给服务恢复时间，避免雪崩。

### 换参数重试（适合参数错误）

当工具返回参数错误时，模型需要修正参数。这通过 `is_error: true` 的反馈实现——模型看到错误描述后，在下一轮推理中生成修正后的参数：

```python
# 错误信息要说清楚怎么修复，让模型能自行纠正
if not os.path.isabs(path):
    return ToolResult(
        content="路径必须是绝对路径，例如 /home/user/file.txt。当前传入的是相对路径：" + path,
        is_error=True
    )
```

## 降级策略

工具失败后，可以换工具或换方法完成相同目标。

### 换工具降级

```text
场景：代码编辑
主路径：Edit（精确修改某几行）
降级：Write（整个文件重写）
原因：Edit 失败时（文件不存在、目标行已变化），改写整个文件往往可行

场景：代码搜索
主路径：LSP（语言服务器，精确符号查询）
降级：Grep（文本搜索）
原因：LSP 服务未启动或不支持当前语言时，grep 几乎总是可用

场景：网络搜索
主路径：web_search（结构化搜索结果）
降级：web_fetch（直接读取特定 URL）
原因：搜索服务超时时，如果已知 URL，直接 fetch 更可靠
```

Claude Code 就有这类隐式降级逻辑：当 `Edit` 工具因目标行不匹配失败，agent 会退回到 `Write` 重写整个文件。这是工程上的务实选择——宁可多消耗一点资源，也要完成任务。

### 跳过非关键步骤

```python
# DeerFlow 风格：非关键工具失败时继续，不中断整体任务
try:
    enrichment = await enrich_context_tool(context)
except ToolError:
    # 上下文丰富化失败不影响主任务
    enrichment = None
    logger.warning("Context enrichment failed, continuing without it")

# 用 enrichment（可能为 None）继续后续步骤
result = await main_task_tool(context, enrichment=enrichment)
```

判断依据：这个步骤失败后，任务的核心目标还能完成吗？如果能，就跳过；如果不能，就上升为需要处理的错误。

### 系统级降级示例

```python
# OpenHarness 风格：工具执行的完整错误处理路径
async def _execute_tool_call(self, tool_name: str, arguments: dict):
    # 1. 工具存在性检查
    tool = self.registry.get(tool_name)
    if tool is None:
        return ToolResult(
            content=f"工具 '{tool_name}' 不存在。可用工具：{list(self.registry.keys())}",
            is_error=True
        )

    # 2. 参数验证（Pydantic 负责，类型/格式错误在这里抓）
    try:
        validated = tool.input_model.model_validate(arguments)
    except ValidationError as e:
        return ToolResult(
            content=f"参数验证失败：{e.errors()}",
            is_error=True
        )

    # 3. 权限检查
    if not self.permission_manager.can_execute(tool, validated):
        return ToolResult(
            content="权限不足，该操作需要用户确认。",
            is_error=True
        )

    # 4. 执行（暂时性错误在这里重试）
    try:
        return await tool.execute(validated)
    except TemporaryError as e:
        # 重试逻辑
        return await self._retry(tool, validated, original_error=e)
    except Exception as e:
        return ToolResult(content=f"工具执行失败：{str(e)}", is_error=True)
```

## 用户确认

某些失败不应该自动处理，而是需要人来决策。

**何时请求用户确认：**
- 权限被拒：操作超出当前授权范围
- 破坏性操作前确认：删除、覆盖、不可逆变更（即使没有失败）
- 模糊的需求：工具要求的输入模型不唯一，需要用户选择

**实现方式：**

工具返回特殊的"需要确认"状态，让 agent 暂停并询问用户：

```python
# Claude Code 的权限流程（简化版）
async def execute_tool(tool_name: str, args: dict, context: ToolUseContext):
    tool = get_tool(tool_name)

    # 需要用户确认的工具
    if not tool.is_read_only() and context.permission_mode == "default":
        approved = await context.request_user_approval(
            tool_name=tool_name,
            args=args,
            description=f"即将执行 {tool_name}，请确认"
        )
        if not approved:
            return ToolResult(
                content="用户拒绝了该操作。",
                is_error=False  # 这不是错误，是用户的主动决定
            )

    return await tool.execute(args)
```

注意：用户拒绝不是 `is_error`。错误是工具本身的失败；用户拒绝是正常的授权流程结果，模型应该接受并继续规划。

## 超时处理

长时间运行的工具需要设置合理的超时时间，而不是无限等待。

```python
import asyncio

async def execute_with_timeout(tool_fn, args, timeout_seconds=30):
    try:
        return await asyncio.wait_for(
            tool_fn(args),
            timeout=timeout_seconds
        )
    except asyncio.TimeoutError:
        return ToolResult(
            content=f"工具执行超时（{timeout_seconds}s）。如果是长时间任务，考虑用后台执行模式。",
            is_error=True
        )
```

**长任务转后台**：有些任务（编译、测试套件、数据处理）本来就需要几分钟。设计上有两种处理方式：
1. 异步执行 + 轮询状态：提交任务后立即返回任务 ID，模型用另一个工具查询状态
2. 流式输出：工具边执行边返回中间结果，让 agent 知道进度

Claude Code 的 `interruptBehavior()` 接口（`cancel` vs `block`）就是在声明：用户按 Ctrl+C 时，这个工具应该立即中止还是等它跑完？这对用户体验有很大影响。

## 熔断：避免无效循环

当工具连续失败多次时，应该停止重试，而不是无限循环：

```python
# 在 agent loop 里检测重复失败
def check_for_loop(tool_call_history: list, window=3) -> bool:
    """检测最近 N 次是否在重复调用相同工具+相同参数"""
    if len(tool_call_history) < window:
        return False
    recent = tool_call_history[-window:]
    # 最近 N 次调用完全相同 → 陷入循环
    return len(set(str(call) for call in recent)) == 1

# 触发熔断时的处理
if check_for_loop(history):
    return AgentResult(
        content="工具连续失败，任务无法继续。最后一个错误：" + last_error,
        status="failed"
    )
```

这是 ReAct 循环的"无限循环"陷阱的工具层解法。

## 三个项目的错误处理对比

| 维度 | Claude Code | OpenHarness | DeerFlow |
|------|------------|-------------|----------|
| 错误回传格式 | tool_result + is_error | ToolResult(is_error) | LangChain ToolMessage |
| 参数验证 | Zod（TypeScript） | Pydantic model_validate | Pydantic + LangChain |
| 重试机制 | 无内置重试，由 agent loop 决策 | 无内置重试 | 无内置重试 |
| 降级策略 | Edit → Write 降级（隐式） | 无，由调用方负责 | 有 fallback tool 概念 |
| 用户确认 | checkPermissions()，三模式 | permission_manager 权限检查 | GuardrailMiddleware 拦截 |
| 超时控制 | interruptBehavior() 声明 | asyncio timeout（工具层） | LangGraph timeout |
| 熔断 | max_turns 全局兜底 | 无 | 无 |

**关键观察：** 三个项目都没有内置的"智能重试"机制——失败信息通过 `is_error` 回传给模型，由模型自己决定下一步策略（修正参数、换工具、还是放弃）。这是正确的设计：工具层负责如实报告错误，决策权在模型。

## 系统 Prompt 里的错误处理指令

工具错误处理不只是代码问题，也需要 prompt 层面的配合。告诉模型如何响应错误：

```text
## 工具失败处理规则

1. 参数错误（is_error=true，错误信息提到参数格式）：
   - 阅读错误信息，修正参数后重试一次
   - 如果仍然失败，报告错误并停止

2. 资源不存在（文件不存在、URL 404）：
   - 先确认这是路径错误还是资源确实不存在
   - 如果是路径错误，修正路径重试
   - 如果资源确实不存在且需要创建，先创建再继续

3. 权限被拒：
   - 不要自动重试
   - 告知用户需要什么权限，询问是否授权

4. 连续 3 次相同工具+相同参数失败：
   - 停止重试
   - 说明遇到的障碍，提出替代方案，询问用户意见

5. 工具返回内容过大：
   - 切换到分段读取
   - 先读前 100 行，判断是否需要继续读取
```

## 常见踩坑

**1. 错误信息不告诉模型怎么修复**

`"Error: invalid input"` ——模型不知道什么无效、怎么修正。好的错误信息应该说："path 参数应该是绝对路径，当前传入的 './file.txt' 是相对路径，请改为 '/home/user/file.txt'"。

**2. 对所有错误都重试**

权限错误、参数格式错误重试没有意义——相同的请求还是相同的错误。要先分类，再决定是否重试。

**3. 忘记设 is_error**

错误内容混入正常返回，模型把"FileNotFoundError: [Errno 2]..."当成有用的工具输出来处理。

**4. 没有最大重试次数**

重试循环没有上限，网络持续故障时 agent 死循环消耗大量时间和 token。

**5. 错误吞没**

```python
try:
    result = await tool.execute(args)
except Exception:
    return None  # 吞掉了异常，模型不知道发生了什么
```

工具失败必须返回带错误描述的 ToolResult，不能 return None 或静默失败。

## 来源

- **Claude Code 2.1.88** — `src/Tool.ts`：`interruptBehavior()` 超时/中断语义；`src/services/tools/toolOrchestration.ts`：批处理执行引擎错误收集与回放；`src/hooks/useCanUseTool.tsx`：权限拒绝时的用户确认流程
- **OpenHarness 0.1.0** — `tools/base.py`：`_execute_tool_call()` 统一错误处理路径，Pydantic 参数验证与错误报告，`is_error` 返回格式
- **DeerFlow 2.0** — `deerflow/guardrails/builtin.py`：GuardrailMiddleware 拦截层，allow/deny 结果与错误原因回传；fallback tool 降级概念

关联 wiki：[[wiki/tool-system]] · [[wiki/query-loop]]
关联模式：[[tool-design-principles]] · [[permission-model]]
