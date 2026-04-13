---
pattern: structured-output
category: output
tags: [json, xml, function-calling, tool-use, parsing, data-extraction, agent-core]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: beginner
related_wiki: [wiki/tool-system, wiki/prompt-system]
related_patterns: [react, system-prompt-design, few-shot]
---

# Structured Output

> 约束模型输出为可预测的结构化格式（JSON/XML/Schema），让程序能可靠解析 LLM 回复，是 tool calling 和数据管道的基础。

## 本质

LLM 默认输出自由文本——流畅、自然，但程序读不了。你想从一段文字里提取出"用户的城市是北京，预算是 5000 元"，靠字符串截取太脆弱，靠正则太费劲。Structured Output 做的事情很简单：在 prompt 层面告诉模型"你的输出不是写给人看的，是写给程序用的，请用 JSON/XML 等格式"。

这本质上是一个翻译层——把 LLM 的自然语言能力转换成程序世界能接受的数据格式。有了这层，LLM 就能成为数据管道的一环，而不只是一个聊天框。

Tool calling 是 Structured Output 的最高形态：模型不只输出数据，还告诉程序"用这个数据去调用哪个函数"，让 LLM 从被动回答变成主动驱动程序执行。

## 什么时候用

**Tool calling / Function calling** — Agent 需要决定调用哪个工具、传什么参数，必须用结构化格式让框架解析并执行。

**Agent 间通信** — 一个 Agent 的输出作为另一个 Agent 的输入，JSON 是事实标准的消息格式。

**数据提取** — 从非结构化文本中提取实体、关系、分类标签（姓名、日期、情绪、关键词等）。

**Pipeline 中间步骤** — 模型的输出需要被下游程序进一步处理、路由或存储时。

**API 响应格式化** — 你的后端需要把 LLM 回复包装成特定格式返回给前端或其他系统。

## 什么时候不用

**自由对话和聊天** — 强制输出 JSON 会让回复变得冷硬，伤害用户体验。

**创意写作** — 格式约束会压制表达自由，不适合故事生成、诗歌等场景。

**输出本身就是自然语言** — 摘要、翻译、解释，这些任务的交付物就是自然语言，没有必要再包一层 JSON。

**原型验证阶段** — 快速验证想法时，先用自然语言输出跑通逻辑，稳定后再加格式约束。

## 模板

### 基础版

**Prompt 指令约束（最简单）** — 直接在 prompt 里要求 JSON 格式，适合快速原型，稳定性一般：

```text
从以下文本中提取用户信息，以 JSON 格式输出，不要输出任何其他内容：

文本：{{user_text}}

输出格式：
{
  "name": "用户姓名",
  "city": "所在城市",
  "budget": 数字类型的预算金额
}
```

关键点：明确说"不要输出任何其他内容"，避免模型在 JSON 前后加解释性文字。

---

**XML 标签约束（Claude 推荐）** — Claude 对 XML 格式有更好的原生理解，解析也更稳定：

```text
分析以下产品评论，用 XML 标签标注结果。

评论：{{review_text}}

<analysis>
  <sentiment>positive|negative|neutral</sentiment>
  <score>1-5 的整数评分</score>
  <key_points>
    <point>核心观点 1</point>
    <point>核心观点 2</point>
  </key_points>
  <summary>一句话总结</summary>
</analysis>
```

优势：XML 天然支持嵌套和列表，不需要担心 JSON 中的转义问题；Claude 在训练中见过大量 XML，遵从率高。

---

**JSON Schema 约束（严格版）** — 在 prompt 中嵌入 Schema，明确每个字段的类型和约束：

```text
你是一个信息提取助手。从用户输入中提取结构化数据，严格遵守以下 JSON Schema，不输出 Schema 之外的任何字段：

Schema:
{
  "type": "object",
  "properties": {
    "event_name": {"type": "string", "description": "事件名称"},
    "date": {"type": "string", "format": "YYYY-MM-DD", "description": "事件日期"},
    "participants": {
      "type": "array",
      "items": {"type": "string"},
      "description": "参与者姓名列表"
    },
    "location": {"type": "string", "nullable": true, "description": "地点，未提及则为 null"}
  },
  "required": ["event_name", "date", "participants"]
}

用户输入：{{input}}

输出仅包含 JSON，不要加 markdown 代码块或任何解释。
```

---

### API 原生 JSON Mode

**OpenAI `response_format`** — API 层面强制 JSON，比 prompt 约束可靠得多：

```python
response = client.chat.completions.create(
    model="gpt-4o",
    response_format={"type": "json_object"},  # 开启 JSON Mode
    messages=[
        {"role": "system", "content": "你是数据提取助手，所有输出必须是合法 JSON。"},
        {"role": "user", "content": f"提取以下文本的关键信息：{text}"}
    ]
)
result = json.loads(response.choices[0].message.content)
```

注意：JSON Mode 只保证输出是合法 JSON，不保证字段结构符合你的预期。配合 prompt 中的 Schema 描述效果更好。

**OpenAI Structured Outputs（更强）** — 传入完整 JSON Schema，API 保证字段结构：

```python
from pydantic import BaseModel

class ExtractedInfo(BaseModel):
    name: str
    city: str
    budget: float

response = client.beta.chat.completions.parse(
    model="gpt-4o-2024-08-06",
    messages=[...],
    response_format=ExtractedInfo,  # 直接传 Pydantic 模型
)
result = response.choices[0].message.parsed  # 直接拿到 Python 对象
```

---

### Agent 集成版：Tool Definition

这是 Structured Output 最核心的应用场景——让模型决定调用哪个工具、传入什么参数。

**Anthropic tool_use 格式：**

```python
tools = [
    {
        "name": "get_current_weather",
        "description": "获取指定城市的当前天气。当用户询问天气相关问题时调用此工具。",
        "input_schema": {
            "type": "object",
            "properties": {
                "location": {
                    "type": "string",
                    "description": "城市名称，例如：北京、上海"
                },
                "unit": {
                    "type": "string",
                    "enum": ["celsius", "fahrenheit"],
                    "description": "温度单位，默认使用 celsius"
                }
            },
            "required": ["location"]
        }
    }
]

response = client.messages.create(
    model="claude-opus-4-5",
    max_tokens=1024,
    tools=tools,
    messages=[{"role": "user", "content": "北京今天天气怎么样？"}]
)

# 解析工具调用
if response.stop_reason == "tool_use":
    tool_use = next(b for b in response.content if b.type == "tool_use")
    tool_name = tool_use.name          # "get_current_weather"
    tool_input = tool_use.input        # {"location": "北京", "unit": "celsius"}
```

**多工具路由（Agent 核心）：**

```python
tools = [
    {
        "name": "search_web",
        "description": "搜索互联网获取最新信息。用于需要实时数据、近期新闻、或训练数据截止后的信息。",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {"type": "string", "description": "搜索关键词"},
                "num_results": {"type": "integer", "description": "返回结果数量，默认 5"}
            },
            "required": ["query"]
        }
    },
    {
        "name": "query_database",
        "description": "查询内部产品数据库。用于产品价格、库存、订单信息等内部数据。",
        "input_schema": {
            "type": "object",
            "properties": {
                "sql": {"type": "string", "description": "SQL 查询语句，只允许 SELECT"},
                "table": {"type": "string", "enum": ["products", "orders", "inventory"]}
            },
            "required": ["sql", "table"]
        }
    },
    {
        "name": "send_email",
        "description": "发送邮件。仅在用户明确要求发送邮件时调用，不可自行决定发送。",
        "input_schema": {
            "type": "object",
            "properties": {
                "to": {"type": "string", "description": "收件人邮箱"},
                "subject": {"type": "string"},
                "body": {"type": "string"}
            },
            "required": ["to", "subject", "body"]
        }
    }
]
```

关键设计原则：description 要说清楚"什么时候用"，不只是"做什么"。多工具场景中，清晰的使用边界是模型正确路由的关键。

---

**工具调用 + 结果回传（完整循环）：**

```python
def run_agent(user_message: str):
    messages = [{"role": "user", "content": user_message}]

    while True:
        response = client.messages.create(
            model="claude-opus-4-5",
            max_tokens=1024,
            tools=tools,
            messages=messages
        )

        if response.stop_reason == "end_turn":
            # 模型直接给出最终回答，无需工具
            return response.content[0].text

        if response.stop_reason == "tool_use":
            # 执行所有工具调用
            tool_results = []
            for block in response.content:
                if block.type == "tool_use":
                    result = execute_tool(block.name, block.input)
                    tool_results.append({
                        "type": "tool_result",
                        "tool_use_id": block.id,
                        "content": str(result)
                    })

            # 把工具结果加回对话，继续下一轮
            messages.append({"role": "assistant", "content": response.content})
            messages.append({"role": "user", "content": tool_results})
```

### 变体对比

以下五种方式按可靠性递增、灵活性递减排列：

| 方式 | 实现 | 可靠性 | 适用场景 |
|------|-----|--------|---------|
| Prompt 指令 | "请输出 JSON" | 低 | 原型验证，不在乎偶尔格式错误 |
| XML 标签约束 | `<tag>...</tag>` | 中 | Claude 首选，嵌套数据，避免 JSON 转义 |
| API JSON Mode | `response_format: json_object` | 中高 | OpenAI/Gemini，保证合法 JSON |
| Function Calling | tools 参数 + stop_reason | 高 | Agent 工具调用，所有主流模型支持 |
| Structured Outputs | Pydantic Schema + `.parse()` | 最高 | 生产级数据提取，字段结构必须精确 |

选择原则：原型用 Prompt 指令，生产 Agent 用 Function Calling，高精度数据提取用 Structured Outputs。

## 模型差异

| 模型 | 结构化输出能力 | 注意事项 |
|------|-------------|---------|
| Claude 4 | 原生支持 `tool_use`；对 XML 格式有强偏好，遵从率高；支持 JSON 约束 | 推荐 XML 标签或 tool_use；纯 JSON prompt 约束偶有多余文字输出；tool_use 后 stop_reason 为 `tool_use` |
| GPT-4 / GPT-4o | 支持 `response_format: json_object`（JSON Mode）和 Structured Outputs（`.parse()` + Pydantic）；function calling 成熟稳定 | JSON Mode 需要在 system prompt 中明确要求 JSON，否则 API 报错；Structured Outputs 仅支持部分新版本模型 |
| Gemini 2 | 支持 `response_mime_type: application/json` 和 `response_schema`；function calling 与 OpenAI 格式高度兼容 | `response_schema` 支持 Pydantic 模型；schema 过复杂时偶有字段遗漏，建议保持 schema 精简 |
| 开源模型（LLaMA 3、Qwen 2.5） | 需要框架支持（llama.cpp grammar、vLLM guided decoding、Outlines）才能可靠约束 | 原生 prompt 约束可靠性差；生产环境必须用 grammar-based 约束（如 Outlines/LMFE）；70B+ 效果好于小模型 |

## 常见踩坑

**JSON 被 markdown 代码块包裹** — 现象：模型输出 ` ```json {...} ``` ` 而不是裸 JSON，导致 `json.loads()` 失败。根因：模型的训练偏好在技术内容前加 markdown 格式。避免方式：在 prompt 中明确说"直接输出 JSON，不要加 markdown 代码块，不要加 ` ``` `"；或在解析前先做 strip 处理（保底方案）：
```python
content = response.strip()
if content.startswith("```"):
    content = content.split("```")[1]
    if content.startswith("json"):
        content = content[4:]
result = json.loads(content.strip())
```

**输出被截断导致 JSON 不完整** — 现象：模型输出到一半 token 用完，JSON 结构不完整，解析失败。根因：`max_tokens` 设置不够，或 schema 太复杂导致输出过长。避免方式：根据预期输出大小设置足够的 `max_tokens`；对于大型 JSON，考虑分字段多次提取而不是一次输出全部；生产代码要加 `try/except` 处理解析失败的情况。

**Schema 过复杂，模型选择性遗漏字段** — 现象：Schema 有 20+ 字段，模型只填了其中几个"重要的"，其余留空或不填。根因：模型对复杂 schema 的注意力不够，会优先填"显眼的"字段。避免方式：控制单次提取的字段数量（建议不超过 10 个）；必要字段和可选字段分开，required 列表只放真正必要的；或分批提取，多次调用合并结果。

**Optional 字段歧义** — 现象：可选字段有时输出 `null`，有时输出 `""`，有时直接不出现，下游代码一堆 `KeyError`。根因：模型对"可选"的理解不一致。避免方式：在 schema 描述中明确说"如果信息不存在，输出 null"；在代码中用 `.get("field", None)` 而不是 `["field"]` 访问可选字段。

**格式指令被长上下文稀释** — 现象：对话进行了很多轮后，模型突然不遵守格式要求了。根因：格式指令在 system prompt 开头，随着对话长度增加，模型对开头指令的注意力下降。避免方式：在每个需要结构化输出的 user message 末尾重复格式要求；对于 agent 场景，用 tool_use 而非 prompt 约束，模型对 tools 参数的遵从性不受上下文长度影响。

**工具描述不清导致错误路由** — 现象：多工具场景下，模型频繁选错工具，或在不该调用工具的时候调用了。根因：工具 description 只说"做什么"，没说"什么时候用"。避免方式：description 要包含使用场景（"当用户问 X 时用这个工具"）；对于危险操作（发邮件、删数据），在 description 中明确限制条件（"仅在用户明确要求时才调用"）；在 system prompt 中也补充工具使用指引，两层约束比一层可靠。

## 组合与选择

**Structured Output + CoT** — 先让模型自由推理（`<thinking>` 标签），最后输出结构化结论。推理阶段用自然语言保证质量，最终答案用 JSON/XML 保证可解析性。实现：在 prompt 中要求先完成所有推理，再输出格式化结果。注意不要要求推理本身也结构化，会打断思维流。

**Structured Output + Few-shot** — 在 prompt 中提供 1-3 个完整的"输入 → 格式化输出"示例，帮模型理解期望的格式风格。特别适合字段语义不直观、或字段间有依赖关系的复杂 schema。

**Structured Output + ReAct** — ReAct 本身就依赖 Structured Output：每一步的 Thought/Action/Observation 就是一种结构化格式。tool_use 是最成熟的 ReAct 实现，把"模型决策→工具执行→结果回传"的循环标准化。

## 来源

- OpenAI. *Function Calling*. https://platform.openai.com/docs/guides/function-calling
- OpenAI. *Structured Outputs*. https://platform.openai.com/docs/guides/structured-outputs
- Anthropic. *Tool Use (Function Calling)*. https://docs.anthropic.com/en/docs/build-with-claude/tool-use
- Google. *Function Calling in Gemini API*. https://ai.google.dev/docs/function_calling
- Prompt Engineering Guide (DAIR.AI). *Function Calling with LLMs*. https://www.promptingguide.ai/applications/function_calling
- Prompt Engineering Guide (DAIR.AI). *Function Calling in AI Agents*. https://www.promptingguide.ai/agents/function-calling
- 关联 wiki: [[wiki/tool-system]] [[wiki/prompt-system]]
