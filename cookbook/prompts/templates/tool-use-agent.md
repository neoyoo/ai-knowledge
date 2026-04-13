---
template: tool-use-agent
scenario: 构建带工具调用能力的 Agent
tags: [agent, tool-use, function-calling, multi-tool]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: intermediate
related_patterns: [react, structured-output, system-prompt-design]
---

# Tool Use Agent

> 为 Agent 配备工具调用能力的完整模板，涵盖系统提示词、Anthropic 和 OpenAI 两种工具定义格式、多工具路由策略、工具失败处理。

## 场景描述

当 Agent 需要调用外部工具（搜索、数据库、API、文件系统、代码执行等）时，仅靠 system prompt 中的文字指令不够——你还需要规范的工具定义、清晰的调用策略和健壮的失败处理。

这个模板给出的不只是"填空题"，而是一套可直接复制到代码里的完整方案：system prompt 定义 Agent 何时用工具、用哪个；工具定义 schema 按 Anthropic/OpenAI 两种格式给出；失败处理逻辑覆盖常见的重试、降级、报错场景。

适用于：需要联网搜索的研究助手、操作数据库的数据查询 Agent、执行代码的开发辅助工具、调用多个 API 的集成 Agent。

## 模板

### System Prompt

```text
# Identity
你是 {{AGENT_NAME}}，{{AGENT_ONE_LINE_DESCRIPTION}}。

# Tools
你拥有以下工具：
{{#each TOOLS}}
- **{{TOOL_NAME}}**：{{TOOL_DESCRIPTION}}。适合用于：{{TOOL_USE_CASE}}
{{/each}}

# Tool Use Strategy
工具选择原则：
- 优先使用最具针对性的工具，不要用通用工具解决专用工具能解决的问题
- 如果一个任务需要多个工具，按 {{TOOL_ORDER_RATIONALE}} 的顺序执行
- 工具调用前，先在内部确认：这个调用的目的是什么？预期返回什么？

并发策略：
- 如果多个工具调用相互独立（结果不互相依赖），{{PARALLEL_STRATEGY}}
- 如果工具调用有依赖（B 需要 A 的结果），必须串行执行

退出条件：
- 当 {{EXIT_CONDITION}} 时，停止调用工具，生成最终回复
- 最多调用 {{MAX_TOOL_CALLS}} 次工具，超出后汇总已有信息给出结论

# Tool Failure Handling
- 工具调用失败（网络错误/超时）：{{RETRY_STRATEGY}}，仍失败则 {{RETRY_FALLBACK}}
- 工具返回空结果：{{EMPTY_RESULT_STRATEGY}}
- 工具返回意外格式：记录原始内容，尝试提取有用信息，无法解析时告知用户

# Rules
- {{RULE_1}}
- {{RULE_2}}
- 在最终回复中，说明使用了哪些工具以及依据，不要隐藏工具调用过程
- 不要在没有工具依据的情况下声称"已查询"或"已验证"

# Output Format
{{OUTPUT_FORMAT}}
```

### Tool Definitions（Anthropic 格式）

```python
tools = [
    {
        "name": "{{TOOL_1_NAME}}",
        "description": "{{TOOL_1_DESCRIPTION}}。当 {{TOOL_1_TRIGGER_CONDITION}} 时使用。不要在 {{TOOL_1_ANTI_USE_CASE}} 时使用。",
        "input_schema": {
            "type": "object",
            "properties": {
                "{{PARAM_1_NAME}}": {
                    "type": "{{PARAM_1_TYPE}}",
                    "description": "{{PARAM_1_DESCRIPTION}}"
                },
                "{{PARAM_2_NAME}}": {
                    "type": "{{PARAM_2_TYPE}}",
                    "description": "{{PARAM_2_DESCRIPTION}}",
                    "enum": ["{{ENUM_VALUE_1}}", "{{ENUM_VALUE_2}}"]
                }
            },
            "required": ["{{REQUIRED_PARAM_1}}", "{{REQUIRED_PARAM_2}}"]
        }
    },
    {
        "name": "{{TOOL_2_NAME}}",
        "description": "{{TOOL_2_DESCRIPTION}}",
        "input_schema": {
            "type": "object",
            "properties": {
                "{{PARAM_NAME}}": {
                    "type": "string",
                    "description": "{{PARAM_DESCRIPTION}}"
                }
            },
            "required": ["{{PARAM_NAME}}"]
        }
    }
]
```

### Tool Definitions（OpenAI / Azure OpenAI 格式）

```python
tools = [
    {
        "type": "function",
        "function": {
            "name": "{{TOOL_1_NAME}}",
            "description": "{{TOOL_1_DESCRIPTION}}。当 {{TOOL_1_TRIGGER_CONDITION}} 时使用。",
            "parameters": {
                "type": "object",
                "properties": {
                    "{{PARAM_1_NAME}}": {
                        "type": "{{PARAM_1_TYPE}}",
                        "description": "{{PARAM_1_DESCRIPTION}}"
                    },
                    "{{PARAM_2_NAME}}": {
                        "type": "{{PARAM_2_TYPE}}",
                        "description": "{{PARAM_2_DESCRIPTION}}",
                        "enum": ["{{ENUM_VALUE_1}}", "{{ENUM_VALUE_2}}"]
                    }
                },
                "required": ["{{REQUIRED_PARAM_1}}", "{{REQUIRED_PARAM_2}}"]
            }
        }
    }
]
```

### 工具调用循环（Python 伪代码）

```python
messages = [{"role": "user", "content": user_input}]
tool_call_count = 0
MAX_CALLS = {{MAX_TOOL_CALLS}}

while tool_call_count < MAX_CALLS:
    response = client.messages.create(
        model="{{MODEL_ID}}",
        system=system_prompt,
        messages=messages,
        tools=tools,
        max_tokens={{MAX_TOKENS}}
    )

    # 没有工具调用，直接返回最终回复
    if response.stop_reason == "end_turn":
        return response.content[0].text

    # 处理工具调用
    tool_uses = [b for b in response.content if b.type == "tool_use"]
    if not tool_uses:
        break

    # 执行工具（可并发）
    tool_results = []
    for tool_use in tool_uses:
        try:
            result = execute_tool(tool_use.name, tool_use.input)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tool_use.id,
                "content": result
            })
        except Exception as e:
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tool_use.id,
                "content": f"Error: {str(e)}",
                "is_error": True
            })
        tool_call_count += 1

    # 追加到对话历史
    messages.append({"role": "assistant", "content": response.content})
    messages.append({"role": "user", "content": tool_results})

# 达到上限，生成降级回复
return generate_fallback_response(messages)
```

## 自定义指南

| 占位符 | 说明 | 示例值 |
|--------|------|--------|
| `{{AGENT_NAME}}` | Agent 名称 | `ResearchBot`、`DataQueryBot` |
| `{{AGENT_ONE_LINE_DESCRIPTION}}` | 一句话描述 | `一个能联网搜索并综合分析信息的研究助手` |
| `{{TOOL_NAME}}` / `{{TOOL_DESCRIPTION}}` | 工具名和描述 | `web_search` / `搜索互联网获取最新信息` |
| `{{TOOL_USE_CASE}}` | 工具适用场景（帮模型选对工具） | `需要最新数据、实时价格、近期事件时` |
| `{{TOOL_ORDER_RATIONALE}}` | 多工具时的调用顺序依据 | `先获取数据再分析`、`先搜索再过滤` |
| `{{PARALLEL_STRATEGY}}` | 并发调用策略 | `可以同时调用多个工具`、`依次调用以控制 API 配额` |
| `{{EXIT_CONDITION}}` | 何时停止工具调用 | `已获得足够信息回答用户问题`、`所有子任务完成` |
| `{{MAX_TOOL_CALLS}}` | 单次对话最大工具调用次数 | `5`、`10`（防止无限循环，建议 5-15） |
| `{{RETRY_STRATEGY}}` | 失败后重试策略 | `用相同参数重试一次`、`用不同关键词重试` |
| `{{RETRY_FALLBACK}}` | 重试仍失败时 | `跳过该工具，记录失败，继续其他步骤`、`告知用户工具不可用` |
| `{{EMPTY_RESULT_STRATEGY}}` | 工具返回空结果时 | `换关键词重试`、`告知用户未找到相关信息` |
| `{{TOOL_1_TRIGGER_CONDITION}}` | 工具 description 里的触发条件（帮模型决策） | `需要查询实时数据时`、`用户提供了 ID 需要查详情时` |
| `{{TOOL_1_ANTI_USE_CASE}}` | 工具 description 里的反用例（防误用） | `答案已知时`、`只需要常识推理时` |
| `{{PARAM_1_TYPE}}` | 参数类型 | `string`、`integer`、`boolean`、`array` |
| `{{REQUIRED_PARAM_1}}` | 必填参数名 | `query`、`user_id`、`file_path` |
| `{{MODEL_ID}}` | 模型 ID | `claude-opus-4-5`、`gpt-4o`、`gemini-2.0-flash` |
| `{{MAX_TOKENS}}` | 最大输出 token | `4096`（工具调用响应通常需要足够空间） |

## 使用示例

### 场景：天气查询 + 建议 Agent

**填充后的 System Prompt**

```text
# Identity
你是 WeatherBot，一个能查询实时天气并给出出行建议的助手。

# Tools
你拥有以下工具：
- **get_weather**：查询指定城市的当前天气。适合用于：用户询问天气、规划出行、需要实时气象数据时
- **get_forecast**：查询未来 7 天天气预报。适合用于：用户计划多天行程或询问未来天气时

# Tool Use Strategy
工具选择原则：
- 询问"现在天气"用 get_weather；询问"这周/明天/周末"用 get_forecast
- 两者都需要时先调 get_weather 再调 get_forecast
- 工具调用前，先确认：我有足够的城市信息吗？

并发策略：
- get_weather 和 get_forecast 相互独立，可以同时调用
- 必须先获取城市名，再调用工具（如果用户没给，先问清楚）

退出条件：
- 获得天气数据后，停止调用工具，生成建议回复
- 最多调用 3 次工具，超出后汇总已有信息给出结论

# Tool Failure Handling
- 调用失败：用同一城市名重试一次，仍失败则告知用户天气服务暂时不可用
- 返回空结果：询问用户是否有城市名的其他写法（如中英文切换）
- 返回意外格式：告知用户获取数据时遇到问题，建议直接查询天气 App

# Rules
- 给出天气数据后，必须附带实际可用的出行建议（不能只报数字）
- 不要在没有工具依据的情况下声称"已查询"或"已验证"
- 在最终回复中，说明数据来源时间

# Output Format
天气 + 建议格式：
**{城市}当前天气**：{温度}°C，{天气状况}，{风速}
**建议**：{1-2 句出行建议}
```

**Tool Definitions（Anthropic 格式）**

```python
tools = [
    {
        "name": "get_weather",
        "description": "查询指定城市的当前实时天气。当用户询问现在的天气状况时使用。不要在用户只询问未来天气预报时使用。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，支持中文或英文，如 '北京' 或 'Beijing'"
                },
                "units": {
                    "type": "string",
                    "description": "温度单位",
                    "enum": ["celsius", "fahrenheit"]
                }
            },
            "required": ["city"]
        }
    },
    {
        "name": "get_forecast",
        "description": "查询指定城市未来 7 天的天气预报。当用户询问明天、这周或特定日期的天气时使用。",
        "input_schema": {
            "type": "object",
            "properties": {
                "city": {
                    "type": "string",
                    "description": "城市名称，支持中文或英文"
                },
                "days": {
                    "type": "integer",
                    "description": "预报天数，1-7 之间"
                }
            },
            "required": ["city"]
        }
    }
]
```

**User Prompt**

```text
北京这周末天气怎么样？适合去爬山吗？
```

**期望输出（工具调用后）**

```
**北京周末天气预报**：
- 周六：18°C，晴转多云，微风 3 级
- 周日：15°C，小雨，东风 2 级

**建议**：周六适合爬山，空气清新，注意带薄外套；周日有小雨，建议推迟户外活动或备好雨具。山顶温度比市区低 3-5°C，两天都建议多带一层衣物。

*数据来源：2026-04-14 10:23 查询*
```

## 适配建议

**工具数量建议**

| 工具数 | 建议 |
|--------|------|
| 1-3 个 | 在 system prompt 详细说明每个工具的触发场景，模型能自主选择 |
| 4-8 个 | 在工具 description 里加上"不要在 X 时使用"的反例，防止工具误用 |
| 8 个以上 | 考虑工具分组（按功能域），或实现动态工具加载（按上下文只暴露相关工具） |

**Claude 4**

工具 description 里用 XML 标签可以提升遵循度：

```python
"description": "<purpose>搜索互联网获取最新信息</purpose><use_when>需要实时数据时</use_when><avoid_when>答案已知时</avoid_when>"
```

开启 prompt caching 可以缓存工具定义，多轮对话时减少重复计算成本（工具定义通常较长，效果显著）。

**GPT-4o**

`tool_choice` 参数控制工具调用行为：
- `"auto"`（默认）：模型自主决定是否调用工具
- `{"type": "function", "function": {"name": "specific_tool"}}`：强制调用特定工具
- `"required"`：强制必须调用工具（防止模型直接回答）

**并发工具调用**

Anthropic API 支持在单次响应中返回多个 `tool_use` 块。如果 system prompt 里说明了哪些工具可以并发，模型会主动返回多个工具调用，可以用 `asyncio.gather` 并行执行：

```python
import asyncio

async def execute_parallel(tool_uses):
    tasks = [execute_tool_async(t.name, t.input) for t in tool_uses]
    return await asyncio.gather(*tasks, return_exceptions=True)
```

**工具失败降级策略**

| 失败场景 | 建议策略 |
|----------|----------|
| 网络超时 | 重试 1 次（不同超时设置），失败告知用户 |
| 返回空结果 | 换参数重试（如不同关键词、更宽的过滤条件） |
| 返回格式错误 | 尝试提取，失败则记录原始响应供调试，用"工具返回异常"告知用户 |
| 权限错误 | 不重试，直接告知用户需要授权，给出授权引导 |
| 超过调用上限 | 汇总已有信息，说明因调用次数限制无法获取更多数据 |

**开源模型（Llama 3 / Qwen 2.5）**

工具调用格式和 Anthropic/OpenAI 不完全兼容，建议：
- 用 few-shot 示例展示工具调用的 JSON 格式
- 将工具列表序列化为文本注入 system prompt，不依赖 native function calling
- 用 structured output（JSON mode）代替原生工具调用，自己解析并执行

## 关联

**使用的 Pattern**
- [[react]] — 本模板的工具调用循环是 ReAct 模式的实现，Thought → Action → Observation 三步
- [[structured-output]] — 工具 input_schema 本质上是 JSON Schema，参见 structured output 的 schema 写法
- [[system-prompt-design]] — 本模板扩展了 agent-system-prompt 的 Capabilities 和 Workflow 模块

**相关 Wiki**
- [[wiki/tool-system]] — 工具系统架构原理，工具注册、路由、执行全链路
- [[wiki/query-loop]] — Agent 循环设计，工具调用在循环中的位置
- [[wiki/context-management]] — 多轮工具调用时的上下文管理策略

**基础模板**
- [[agent-system-prompt]] — 不需要工具调用时的精简版 Agent system prompt
