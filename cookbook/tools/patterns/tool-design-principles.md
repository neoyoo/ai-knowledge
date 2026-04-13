---
pattern: tool-design-principles
category: tool-design
tags: [design, naming, description, parameters, schema, best-practices]
sources: [claude-code, deer-flow, openharness]
related_wiki: [wiki/tool-system]
related_patterns: [error-handling, permission-model]
---

# Tool Design Principles

> 怎样设计一个 LLM 能正确理解和使用的工具——从命名到参数到描述的全套原则。

## 本质

工具不是给人用的，是给 LLM 看的。人类程序员读 API 文档，有上下文，能猜意图，能实验。LLM 在推理时只有你给的 schema 描述，没有第二次机会澄清。

这意味着：**工具 schema 的质量直接决定模型调用的准确率**。一个描述含糊的工具，模型会选错、传错参数、在不该用的时候用；一个设计清晰的工具，模型几乎不会出错。

与 REST API 设计不同，工具设计的主要受众是 LLM，次要受众才是维护代码的工程师。

## 命名原则

**动词_名词，一眼知道在做什么**

```text
好：read_file、write_file、search_web、execute_bash
坏：file_operation、process、handle_request、do_thing
```

规则：
- 用动词开头：`read_`、`write_`、`search_`、`execute_`、`list_`、`create_`、`delete_`
- 名词部分要具体：`file`（不是 `resource`）、`bash`（不是 `command`）
- 用下划线分隔（snake_case），不用驼峰、不用连字符
- 名称与功能一一对应——避免一个工具干多件事然后取一个模糊的名字

**不要用缩写**：`fs_op` 不如 `read_file`，`exec` 不如 `execute_bash`。模型训练集中完整单词出现频率远高于缩写，解码更稳定。

**粒度宁小勿大**：与其设计一个 `file_operation(action: "read"|"write"|"delete", ...)` 大而全的工具，不如拆成 `read_file`、`write_file`、`delete_file` 三个。拆分后：
- 模型选工具时语义更明确
- 权限可以精确到读/写/删三个级别
- 只读工具可以并行，写工具串行，分开后更好调度

Claude Code 就是这样做的：`Read`、`Write`、`Edit` 是三个独立工具，而不是 `FileOperation(mode)`。

## 描述原则

**说"做什么"，不说"怎么做"**

工具描述是 LLM 选工具时的依据，它需要回答两个问题：
1. 这个工具是干什么用的？
2. 什么时候应该用它（vs 其他工具）？

```python
# 坏的描述——说了怎么做，但没说什么时候用
"description": "调用 subprocess 模块执行系统命令，返回 stdout 和 stderr"

# 好的描述——说了做什么、用哪里、不用哪里
"description": "在 shell 中执行任意 bash 命令。适用于：运行脚本、操作文件系统、安装包、查看系统状态。不适用于：需要持久 shell 状态（cd 改变目录后下次调用不保留）。"
```

**必须包含：**
- 功能描述（动作 + 对象）
- 典型使用场景 1-2 个
- 什么情况下**不应该**用（与近邻工具的边界）

**长度控制**：20-50 字是最佳区间。超过 100 字模型容易抓不住重点；少于 10 字又太模糊。每个工具描述消耗 tokens，100 个工具的系统里描述太长直接影响 context 预算。

OpenHarness 把 `description` 作为 `BaseTool` 的必填字段，没有描述就无法实例化工具——强制执行设计纪律。

DeerFlow 的每个工具调用还额外携带一个 `description` 参数，让模型在调用时说明本次调用的**意图**（"为什么在这里调用这个工具"）。这为调试日志提供了极大价值，也帮助 guardrail 做语义审查。

## 参数设计

**必填 vs 可选要明确**

JSON Schema 的 `required` 数组控制必填字段。原则：
- 工具执行不可缺少的参数放 `required`
- 有合理默认值的参数放可选，并在描述里写明默认值
- 不要把"通常不需要改"的参数也设成必填——每个必填参数都是模型的负担

```python
# OpenHarness 示例（Python Pydantic 风格）
class ReadFileInput(BaseModel):
    path: str = Field(description="要读取的文件路径，支持绝对路径和相对于工作目录的相对路径")
    encoding: str = Field(default="utf-8", description="文件编码，默认 utf-8")
    start_line: Optional[int] = Field(default=None, description="从第几行开始读取，None 表示从头读")
    end_line: Optional[int] = Field(default=None, description="读到第几行结束，None 表示读到文件末尾")
```

**用 enum 限制选择范围**

当参数只有固定几个合法值时，用 `enum` 声明，不要让模型自由发挥：

```json
{
  "name": "sort_order",
  "type": "string",
  "enum": ["asc", "desc"],
  "description": "排序方向：asc 升序，desc 降序"
}
```

没有 `enum` 约束时，模型可能传 `"ascending"`、`"ASCENDING"`、`"up"` 等各种变体，引发解析失败。

**description 字段不可省**

每个参数的 `description` 都必须填写，特别是：
- 说明参数的**格式**（绝对路径 vs 相对路径？ISO 时间格式还是 Unix 时间戳？）
- 说明参数的**边界**（最大长度？支持哪些特殊字符？）
- 说明参数的**关系**（`start_line` 和 `end_line` 同时省略意味着什么？）

省略 `description` 是最常见的"模型老是传错参数"的根本原因。

**类型越具体越好**

| 能用具体类型 | 不要用 |
|------------|--------|
| `integer` (行号、数量) | `number` |
| `array of string` (文件路径列表) | `string` (逗号分隔) |
| `boolean` (是否覆盖) | `string` ("true"/"false") |
| `object` with named fields | `string` (JSON 字符串) |

模型见到 `string` 类型会自由发挥格式；见到 `integer` 或 `boolean` 知道确切期望，格式错误率大幅下降。

## 返回值设计

**结构化 vs 纯文本**

工具的返回值会进入消息历史，模型需要从中提取信息。设计原则：

- **成功时**：结构化数据（JSON）优于纯文本，关键字段明确命名
- **失败时**：错误信息必须**可操作**，告诉模型怎么修复

```python
# 坏的错误返回
"Error: file not found"

# 好的错误返回
"Error: 文件 /tmp/output.txt 不存在。可能原因：(1) 路径错误，请确认路径是否包含空格或特殊字符；(2) 文件尚未创建，请先用 write_file 工具创建该文件。"
```

可操作的错误信息能让模型自行修复而不是卡住、或产生无意义的重试。

**控制输出大小**

工具输出会消耗 context 预算。设计时要考虑：
- 大文件 `read_file` 要支持 `start_line`/`end_line` 分段读取
- 搜索结果返回前 N 条，不要全量返回
- 超长输出考虑"摘要 + 完整内容写入临时文件"的模式

Hermes Agent 的三层防溢出方案：工具内截断 → 单结果持久化到文件 → Turn 级预算聚合，是生产环境可参考的完整实现。

**is_error 标记**

Anthropic tool_use API 支持在 `tool_result` 里设置 `is_error: true`，明确告知模型这是错误返回而不是正常内容。模型看到 `is_error` 会切换到错误处理推理模式，比让模型自己判断"这段文字是不是错误"更可靠。

```python
# 工具调用失败时的返回格式
{
    "type": "tool_result",
    "tool_use_id": "tool_123",
    "content": "文件不存在：/tmp/output.txt",
    "is_error": True  # 关键字段
}
```

## 只读标记与并发安全

**声明只读是并发调度的基础**

只读工具（不修改任何外部状态）可以安全并行执行；写工具需要串行，避免竞态条件。这个声明应该由工具自己做，而不是让调度层猜。

Claude Code 的 `isConcurrencySafe()` 和 OpenHarness 的 `is_read_only()` 都体现了这个思路：

```typescript
// Claude Code 风格（TypeScript）
class ReadTool implements Tool {
    isConcurrencySafe(): boolean {
        return true;  // 纯读操作，可以并发
    }
}

class WriteTool implements Tool {
    isConcurrencySafe(): boolean {
        return false;  // 写操作，串行执行
    }
}
```

```python
# OpenHarness 风格（Python）
class ReadFileTool(BaseTool):
    def is_read_only(self, arguments: dict) -> bool:
        return True

class WriteFileTool(BaseTool):
    def is_read_only(self, arguments: dict) -> bool:
        return False
```

**Fail-closed 默认策略**：Claude Code 把未声明 `isConcurrencySafe` 的工具默认视为不安全，只有显式声明安全才并发。这比 fail-open（默认并发）更保守，但能防止意外的并发写入。

## 三个项目的实际对比

| 维度 | Claude Code | OpenHarness | DeerFlow |
|------|------------|-------------|----------|
| 工具基类 | TypeScript interface `Tool` | Python `BaseTool` abstract class | Python `BaseTool` + LangChain Tool |
| Schema 生成 | 手写 Zod schema → JSON Schema | Pydantic `model_json_schema()` 自动生成 | Pydantic + 手写 description |
| 并发安全声明 | `isConcurrencySafe()` 方法 | `is_read_only()` 方法 | 无显式声明，均串行 |
| 调用意图记录 | 无 | 无 | `description` 参数（每次调用说明意图） |
| 命名风格 | PascalCase（`ReadFile`、`BashTool`） | snake_case（`read_file`、`bash_execute`） | snake_case（`read_file`、`web_search`） |
| 工具粒度 | 细粒度（Read/Write/Edit 分开） | 细粒度 | 中等粒度（read_file/write_file 分开，但 bash 合一） |

## 完整工具定义示例

```python
# Python + Pydantic（OpenHarness 风格，直接可用）
from pydantic import BaseModel, Field
from typing import Optional
from tools.base import BaseTool

class SearchWebInput(BaseModel):
    query: str = Field(
        description="搜索查询词。使用关键词而非完整问句效果更好，例如 'Python asyncio tutorial' 而非 '怎么用 Python 写异步代码'"
    )
    num_results: int = Field(
        default=5,
        description="返回结果数量，范围 1-20，默认 5。任务需要广度时用 10-20，需要精准时用 3-5"
    )
    language: str = Field(
        default="en",
        description="搜索语言代码，例如 'en'（英文）、'zh'（中文）。中文技术文档质量差时建议用 'en'"
    )

class SearchWebTool(BaseTool):
    name = "search_web"
    description = (
        "搜索互联网获取最新信息。"
        "适用于：查询实时数据（股价、新闻、天气）、训练截止日期后的信息、特定产品文档。"
        "不适用于：数学计算、代码分析、文件读写——这些有专用工具。"
    )
    input_model = SearchWebInput

    def is_read_only(self, arguments: dict) -> bool:
        return True  # 只读，可并行

    async def execute(self, validated_input: SearchWebInput) -> str:
        # 实现细节...
        pass
```

## 常见踩坑

**1. 工具名太泛**

`process_request`、`handle_data`、`do_operation` ——模型看不懂，会随机选或跳过。

**2. description 只有一句话但没说边界**

"读取文件内容"——模型不知道文件太大怎么办、不知道二进制文件怎么处理、不知道文件不存在返回什么。

**3. 参数类型用 string 图省事**

`action: "read"|"write"|"delete"` 用 string 类型而不加 enum，导致模型传 `"READ"`、`"读取"`、`"r"` 等各种变体。

**4. 错误返回不可操作**

`"Error: permission denied"` ——模型不知道应该改权限、换路径、还是请求用户确认。

**5. 一个工具干太多事**

`file_manager(action, path, content, encoding, mode...)` ——参数爆炸，模型难以准确填写所有字段组合；权限控制也无法精确到读/写/删三个级别。

## 来源

- **Claude Code 2.1.88** — `src/Tool.ts`：工具协议接口，`isConcurrencySafe()`、`interruptBehavior()` 设计；`src/tools.ts`：分层工具池架构
- **OpenHarness 0.1.0** — `tools/base.py`：`BaseTool` 抽象类，`to_api_schema()`、`is_read_only()` 实现；Pydantic 一套定义兼顾验证和 Schema 生成
- **DeerFlow 2.0** — 工具调用 `description` 参数（意图记录）设计；`GuardrailMiddleware` 中对工具描述的语义审查

关联 wiki：[[wiki/tool-system]] · [[wiki/query-loop]]
关联模式：[[error-handling]] · [[permission-model]]
