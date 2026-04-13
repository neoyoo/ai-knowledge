---
template: data-extraction
scenario: 从非结构化文本提取结构化数据
tags: [extraction, json, parsing, data-pipeline, etl]
models_tested: [claude-4, gpt-4, gemini-2]
difficulty: beginner
related_patterns: [structured-output, few-shot, prompt-chaining]
---

# 数据提取模板

> 从非结构化文本（合同、报告、日志、邮件）按 JSON Schema 提取结构化字段，缺失字段输出 null，不猜测。

## 场景描述

发票解析、合同信息提取、简历结构化、客服工单分类——这类任务的共同点是：输入是自由文本，输出必须是固定结构，用于入库或下游系统消费。这个模板提供了单实体提取和批量多实体提取两种变体，以及 JSON Schema 定义规范和缺失字段处理策略，可直接集成进 ETL 管道。

## 模板

### 变体一：单实体提取

适合从一段文本中提取一个结构化对象（如一份合同、一张发票）。

**System Prompt**

```
你是一个数据提取引擎。从用户提供的文本中，严格按照指定 JSON Schema 提取信息。

提取规则：
1. 只提取文本中明确出现的信息，禁止推断、补全或根据常识填充
2. 字段在文本中不存在时，输出 null，不要省略该字段
3. 日期统一格式化为 ISO 8601（YYYY-MM-DD）
4. 金额统一为数字类型（不含货币符号），货币类型单独存字段
5. 只输出 JSON，不要任何解释、前缀或 Markdown 围栏
6. 输出必须是合法 JSON，可直接被 JSON.parse() 解析

JSON Schema：
{{json_schema}}
```

**User Prompt**

```
请从以下文本提取数据：

{{source_text}}
```

---

### 变体二：批量多实体提取

适合从一段文本中提取多个同类型实体（如从一篇文章中提取所有人物、从订单列表中提取所有商品行）。

**System Prompt**

```
你是一个批量数据提取引擎。从用户提供的文本中，找出所有符合条件的实体，按照指定 JSON Schema 结构化输出为数组。

提取规则：
1. 识别文本中所有独立的 {{entity_type}} 实例，每个实例作为数组中的一个对象
2. 每个对象只提取文本中明确出现的信息，缺失字段输出 null
3. 日期统一格式化为 ISO 8601（YYYY-MM-DD）
4. 金额统一为数字类型，货币类型单独存字段
5. 如果文本中没有任何符合条件的实体，输出空数组 []
6. 只输出 JSON 数组，不要解释或 Markdown 围栏

单个实体的 JSON Schema：
{{entity_schema}}

输出格式：[{...}, {...}, ...]
```

**User Prompt**

```
请从以下文本中提取所有 {{entity_type}}：

{{source_text}}
```

---

### 变体三：带验证的提取（适合关键字段不可缺失的场景）

当某些字段是必填项，缺失时需要报告错误而非静默输出 null。

**System Prompt**

```
你是一个数据提取引擎，带字段验证功能。

提取规则：
1. 按照 JSON Schema 从文本提取信息
2. 可选字段缺失时输出 null
3. 必填字段（required 标注的）缺失时，不在 data 中输出该字段，而是在 errors 数组中记录
4. 日期格式化为 YYYY-MM-DD，金额为数字类型
5. 只输出 JSON，不要解释

JSON Schema：
{{json_schema}}

输出格式：
{
  "data": { ...提取的字段... },
  "errors": [
    { "field": "字段名", "reason": "未在文本中找到" }
  ]
}
errors 为空时输出空数组 []。
```

**User Prompt**

```
{{source_text}}
```

## 自定义指南

| Placeholder | 说明 | 示例 |
|------------|------|------|
| `{{json_schema}}` | 目标输出的 JSON Schema，包含字段名、类型、描述；见下方 Schema 模板 | `{"type":"object","properties":{...}}` |
| `{{source_text}}` | 待提取的原始文本，支持任意格式（纯文本、HTML、Markdown、PDF 解析后的文字） | "甲方：北京科技有限公司，合同金额：人民币50万元..." |
| `{{entity_type}}` | 批量提取版专用，说明要提取什么类型的实体，用中文或英文均可 | "采购订单行项目" / "人物信息" / "会议事项" |
| `{{entity_schema}}` | 批量提取版专用，单个实体的 Schema，不含外层数组包装 | `{"type":"object","properties":{...}}` |

### JSON Schema 定义模板

```json
{
  "type": "object",
  "properties": {
    "contract_id": {
      "type": ["string", "null"],
      "description": "合同编号，格式通常为字母+数字组合"
    },
    "party_a": {
      "type": ["string", "null"],
      "description": "甲方名称，通常是发起方/买方"
    },
    "party_b": {
      "type": ["string", "null"],
      "description": "乙方名称，通常是承接方/卖方"
    },
    "amount": {
      "type": ["number", "null"],
      "description": "合同总金额，纯数字，不含货币符号"
    },
    "currency": {
      "type": ["string", "null"],
      "description": "货币类型，如 CNY、USD、EUR"
    },
    "sign_date": {
      "type": ["string", "null"],
      "description": "签订日期，ISO 8601 格式 YYYY-MM-DD"
    },
    "expiry_date": {
      "type": ["string", "null"],
      "description": "到期日期，ISO 8601 格式 YYYY-MM-DD"
    }
  },
  "required": ["party_a", "party_b", "amount"]
}
```

**缺失字段处理策略对比**：

| 策略 | 做法 | 适用场景 |
|------|------|---------|
| null 填充（推荐） | 缺失字段输出 `null` | 下游系统期望固定结构，字段存在但为空是合法状态 |
| 省略字段 | 缺失字段不输出 | 下游系统用 `key in obj` 判断字段是否存在，稀疏结构 |
| 错误报告 | 必填字段缺失时写入 errors 数组 | 需要区分"字段确实没有"和"提取失败"，有质量监控需求 |

**推荐用 null 填充**：省略字段会导致下游系统 `obj.field` 返回 undefined，引发隐性 bug。null 明确表示"字段存在但无值"，语义清晰。

## 使用示例

**输入（合同文本）**

```
source_text: """
采购合同
合同编号：PO-2024-00312

甲方（买方）：深圳绿能科技有限公司
乙方（卖方）：上海精密仪器制造有限公司

合同金额：人民币 128,000 元整（税后）
签订日期：2024年3月15日
合同有效期至：2025年3月14日

主要采购内容：精密传感器组件 200 套
"""

json_schema: （见上方 Schema 定义模板）
```

**输出（预期）**

```json
{
  "contract_id": "PO-2024-00312",
  "party_a": "深圳绿能科技有限公司",
  "party_b": "上海精密仪器制造有限公司",
  "amount": 128000,
  "currency": "CNY",
  "sign_date": "2024-03-15",
  "expiry_date": "2025-03-14"
}
```

---

**输入（批量提取，会议纪要）**

```
entity_type: 行动项
source_text: """
2024年Q1项目复盘会议纪要

决议事项：
1. 张伟负责在3月20日前完成性能测试报告，提交给技术委员会审核
2. 产品部门需于3月25日提供下季度需求排期，发送至项目群
3. 李梅跟进供应商合同续签，4月1日前确认结果
"""

entity_schema: {
  "type": "object",
  "properties": {
    "owner": {"type": ["string", "null"], "description": "负责人姓名"},
    "task": {"type": ["string", "null"], "description": "任务描述"},
    "deadline": {"type": ["string", "null"], "description": "截止日期 YYYY-MM-DD"},
    "deliverable": {"type": ["string", "null"], "description": "交付物或交付对象"}
  }
}
```

**输出（预期）**

```json
[
  {
    "owner": "张伟",
    "task": "完成性能测试报告",
    "deadline": "2024-03-20",
    "deliverable": "提交技术委员会审核"
  },
  {
    "owner": "产品部门",
    "task": "提供下季度需求排期",
    "deadline": "2024-03-25",
    "deliverable": "发送至项目群"
  },
  {
    "owner": "李梅",
    "task": "跟进供应商合同续签",
    "deadline": "2024-04-01",
    "deliverable": "确认续签结果"
  }
]
```

## 适配建议

**Schema 中的描述字段是关键** — `description` 不是给人看的注释，是给模型的提取线索。写清楚"这个字段长什么样"、"在文本中通常以什么形式出现"，比字段名本身更有用。"金额，纯数字，不含货币符号"比单纯写"amount"准确率高得多。

**少样本示例显著提升复杂格式** — 如果文本格式不规则（扫描件 OCR 结果、口语化描述、多语言混排），在 system prompt 末尾加 1-2 个 few-shot 示例。格式越混乱，示例的收益越大。

**长文本分段提取** — 超过 4k tokens 的文档直接提取容易丢失中间部分的信息。切分成段落/章节，每段独立提取后合并结果，最后做去重（尤其是批量提取时）。

**验证 JSON 有效性** — 模型偶尔会输出带尾部逗号、注释或截断的无效 JSON。管道中必须有 try/catch 捕获解析错误，失败时记录原始输出用于 debug，不要静默丢弃。

**数字和日期的标准化在 prompt 里做，不要靠后处理** — 明确要求"金额输出纯数字"和"日期输出 YYYY-MM-DD"，让模型直接输出标准格式，省去后处理的正则逻辑和边界 case 处理。

**Claude 4 的工具调用方式更稳定** — 用 tool use / function calling 替代纯文本 JSON 输出，模型被迫按 schema 格式输出，无效 JSON 概率接近零。schema 直接定义在 tool 的 `input_schema` 里，不需要在 prompt 中重复说明。

## 关联

**Patterns**
- [[cookbook/prompts/patterns/structured-output]] — JSON 输出的完整原理、格式控制技巧、tool use vs 纯文本对比
- [[cookbook/prompts/patterns/few-shot]] — 文本格式不规则时，加示例的写法和选例原则
- [[cookbook/prompts/patterns/prompt-chaining]] — 长文档分段提取后，用 chaining 合并和去重多段结果

**Wiki**
- [[wiki/prompt-system]] — system prompt 结构设计、指令优先级、输出格式控制
- [[wiki/tool-system]] — 用 tool use 替代纯文本 JSON 输出的工程实现，schema 定义规范
