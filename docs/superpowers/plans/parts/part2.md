# neoagent v1 实现计划 — Part 2: Tool System

> **For agentic workers:** Use `superpowers:subagent-driven-development` to execute this plan task-by-task.
> Each Task is independently executable. Steps use checkbox (`- [ ]`) syntax for tracking.

**范围**: Task 4 (BaseTool) + Task 5 (ToolRegistry) + Task 6 (PermissionChecker)

**前置依赖**: `neoagent/core/types.py` 已由 Task 2 定义（`ToolCall`, `ToolResult`, `Message` 等类型）

**目录结构**（本 Part 涉及）:
```
neoagent/
└── tools/
    ├── __init__.py
    ├── base.py        # Task 4
    ├── registry.py    # Task 5
    └── permission.py  # Task 6
tests/
└── tools/
    ├── __init__.py
    ├── test_base.py
    ├── test_registry.py
    └── test_permission.py
```

---

## Task 4: Tool 基类 (`neoagent/tools/base.py`)

### Files

- `neoagent/tools/__init__.py`
- `neoagent/tools/base.py`
- `tests/tools/__init__.py`
- `tests/tools/test_base.py`

---

### Step 1 — 写失败测试

文件路径: `tests/tools/test_base.py`

```python
from __future__ import annotations

import pytest
from pydantic import BaseModel

from neoagent.core.types import ToolResult
from neoagent.tools.base import BaseTool


# ---------- 具体 Tool 实现（用于测试） ----------

class AddInput(BaseModel):
    a: int
    b: int


class AddTool(BaseTool):
    name = "add"
    description = "将两个整数相加并返回结果"
    input_model = AddInput
    permission = "auto"
    is_concurrent_safe = True

    async def execute(self, input: AddInput) -> ToolResult:  # type: ignore[override]
        return ToolResult(call_id="", output=str(input.a + input.b))


class DenyTool(BaseTool):
    name = "deny_op"
    description = "被拒绝的危险操作示例工具"
    input_model = AddInput
    permission = "deny"
    is_concurrent_safe = False

    async def execute(self, input: AddInput) -> ToolResult:  # type: ignore[override]
        return ToolResult(call_id="", output="should not run")


# ---------- 测试：实例化 ----------

def test_add_tool_instantiation():
    tool = AddTool()
    assert tool.name == "add"
    assert tool.permission == "auto"
    assert tool.is_concurrent_safe is True


def test_deny_tool_defaults():
    tool = DenyTool()
    assert tool.permission == "deny"
    assert tool.is_concurrent_safe is False


# ---------- 测试：get_schema() 结构 ----------

def test_get_schema_contains_name_and_description():
    tool = AddTool()
    schema = tool.get_schema()
    assert schema["name"] == "add"
    assert schema["description"] == "将两个整数相加并返回结果"


def test_get_schema_contains_input_schema():
    tool = AddTool()
    schema = tool.get_schema()
    # 顶层必须有 input_schema 键（Anthropic API 格式）
    assert "input_schema" in schema
    input_schema = schema["input_schema"]
    assert input_schema["type"] == "object"
    assert "a" in input_schema["properties"]
    assert "b" in input_schema["properties"]


def test_get_schema_required_fields():
    tool = AddTool()
    schema = tool.get_schema()
    required = schema["input_schema"].get("required", [])
    assert "a" in required
    assert "b" in required


def test_get_schema_field_types():
    tool = AddTool()
    schema = tool.get_schema()
    props = schema["input_schema"]["properties"]
    assert props["a"]["type"] == "integer"
    assert props["b"]["type"] == "integer"


# ---------- 测试：execute() ----------

@pytest.mark.asyncio
async def test_execute_returns_tool_result():
    tool = AddTool()
    input_data = AddInput(a=3, b=4)
    result = await tool.execute(input_data)
    assert isinstance(result, ToolResult)
    assert result.output == "7"
    assert result.is_error is False


@pytest.mark.asyncio
async def test_execute_with_negative_numbers():
    tool = AddTool()
    result = await tool.execute(AddInput(a=-5, b=3))
    assert result.output == "-2"


# ---------- 测试：ABC 强制抽象 ----------

def test_cannot_instantiate_base_tool_directly():
    with pytest.raises(TypeError):
        BaseTool()  # type: ignore[abstract]


def test_must_implement_execute():
    class IncompleteTool(BaseTool):
        name = "incomplete"
        description = "不完整的工具，未实现 execute"
        input_model = AddInput

    with pytest.raises(TypeError):
        IncompleteTool()  # type: ignore[abstract]
```

---

### Step 2 — 跑测试（应全部失败）

```bash
cd /path/to/neoagent_project
pytest tests/tools/test_base.py -v 2>&1 | head -40
```

预期：`ModuleNotFoundError: No module named 'neoagent.tools.base'`（或类似导入错误）

---

### Step 3 — 写实现

文件路径: `neoagent/tools/__init__.py`

```python
from __future__ import annotations
```

文件路径: `neoagent/tools/base.py`

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from typing import Literal

from pydantic import BaseModel

from neoagent.core.types import ToolResult


class BaseTool(ABC):
    """
    所有工具的抽象基类。

    子类必须定义：
    - name: 工具名称（Anthropic API 使用，snake_case）
    - description: 工具描述（20-50 字，过长浪费 token）
    - input_model: Pydantic BaseModel 子类（一套定义同时用于验证和 schema 生成）

    可选覆盖：
    - permission: "auto" | "ask" | "deny"（默认 "ask"）
    - is_concurrent_safe: bool（默认 False，fail-closed 安全策略）
    """

    name: str
    description: str
    input_model: type[BaseModel]
    permission: Literal["auto", "ask", "deny"] = "ask"
    is_concurrent_safe: bool = False

    def get_schema(self) -> dict:
        """
        从 input_model 自动生成 Anthropic API 格式的 tool schema。

        返回格式：
        {
            "name": "<tool_name>",
            "description": "<tool_description>",
            "input_schema": {
                "type": "object",
                "properties": {...},
                "required": [...]
            }
        }

        设计原则：
        - Pydantic model_json_schema() 生成原始 JSON Schema
        - 移除 Pydantic 附加的 "title" 等字段（保持 schema 简洁）
        - 将原始 schema 嵌套到 "input_schema" 键（Anthropic API 约定）
        """
        raw_schema = self.input_model.model_json_schema()

        # 清理 Pydantic 生成的顶层 title（不需要传给 API）
        raw_schema.pop("title", None)

        return {
            "name": self.name,
            "description": self.description,
            "input_schema": raw_schema,
        }

    @abstractmethod
    async def execute(self, input: BaseModel) -> ToolResult:
        """
        执行工具逻辑。

        参数:
            input: 已通过 input_model 验证的 Pydantic 实例

        返回:
            ToolResult(call_id, output, is_error)
            - call_id 由 ToolRegistry 在调用时填充
            - 工具实现可保持 call_id="" 占位

        异常处理:
            工具内部异常应捕获并返回 ToolResult(is_error=True)
            不应向外抛出异常（上层 ToolRegistry 会兜底）
        """
        ...
```

---

### Step 4 — 跑测试（应全部通过）

```bash
pytest tests/tools/test_base.py -v
```

预期输出：
```
tests/tools/test_base.py::test_add_tool_instantiation PASSED
tests/tools/test_base.py::test_deny_tool_defaults PASSED
tests/tools/test_base.py::test_get_schema_contains_name_and_description PASSED
tests/tools/test_base.py::test_get_schema_contains_input_schema PASSED
tests/tools/test_base.py::test_get_schema_required_fields PASSED
tests/tools/test_base.py::test_get_schema_field_types PASSED
tests/tools/test_base.py::test_execute_returns_tool_result PASSED
tests/tools/test_base.py::test_execute_with_negative_numbers PASSED
tests/tools/test_base.py::test_cannot_instantiate_base_tool_directly PASSED
tests/tools/test_base.py::test_must_implement_execute PASSED
10 passed in 0.XXs
```

---

### Step 5 — 检查点

- [ ] `BaseTool` 是 ABC，无法直接实例化
- [ ] `get_schema()` 返回 `name` + `description` + `input_schema`（Anthropic API 格式）
- [ ] `input_schema` 包含 `type: "object"` + `properties` + `required`
- [ ] `permission` 默认 `"ask"`，`is_concurrent_safe` 默认 `False`（fail-closed）
- [ ] `execute()` 是抽象方法，子类不实现则实例化抛 `TypeError`
- [ ] 10 个测试全部通过，0 个跳过

---

## Task 5: Tool Registry (`neoagent/tools/registry.py`)

### Files

- `neoagent/tools/registry.py`
- `tests/tools/test_registry.py`

---

### Step 1 — 写失败测试

文件路径: `tests/tools/test_registry.py`

```python
from __future__ import annotations

import asyncio
from typing import ClassVar

import pytest
from pydantic import BaseModel

from neoagent.core.types import ToolCall, ToolResult
from neoagent.tools.base import BaseTool
from neoagent.tools.registry import ToolRegistry


# ---------- 测试用 Tool 定义 ----------

class AddInput(BaseModel):
    a: int
    b: int


class MultiplyInput(BaseModel):
    x: int
    y: int


class SlowInput(BaseModel):
    delay_ms: int = 0


class AddTool(BaseTool):
    name = "add"
    description = "将两个整数相加并返回结果"
    input_model = AddInput
    permission = "auto"
    is_concurrent_safe = True

    async def execute(self, input: AddInput) -> ToolResult:  # type: ignore[override]
        return ToolResult(call_id="", output=str(input.a + input.b))


class MultiplyTool(BaseTool):
    name = "multiply"
    description = "将两个整数相乘并返回乘积"
    input_model = MultiplyInput
    permission = "ask"
    is_concurrent_safe = True

    async def execute(self, input: MultiplyInput) -> ToolResult:  # type: ignore[override]
        return ToolResult(call_id="", output=str(input.x * input.y))


class DangerousTool(BaseTool):
    name = "dangerous_op"
    description = "被完全拒绝的危险操作工具"
    input_model = AddInput
    permission = "deny"
    is_concurrent_safe = False

    async def execute(self, input: AddInput) -> ToolResult:  # type: ignore[override]
        return ToolResult(call_id="", output="never")


class SerialTool(BaseTool):
    """串行工具：记录执行顺序"""
    name = "serial_op"
    description = "必须串行执行的有状态操作工具"
    input_model = AddInput
    permission = "auto"
    is_concurrent_safe = False

    execution_order: ClassVar[list[int]] = []

    async def execute(self, input: AddInput) -> ToolResult:  # type: ignore[override]
        SerialTool.execution_order.append(input.a)
        await asyncio.sleep(0)  # 让出事件循环
        return ToolResult(call_id="", output=str(input.a))


class ErrorTool(BaseTool):
    name = "error_op"
    description = "模拟工具执行失败的测试专用工具"
    input_model = AddInput
    permission = "auto"
    is_concurrent_safe = True

    async def execute(self, input: AddInput) -> ToolResult:  # type: ignore[override]
        raise RuntimeError("tool execution failed")


class LargeOutputTool(BaseTool):
    name = "large_output"
    description = "产生超大输出用于测试结果截断的工具"
    input_model = AddInput
    permission = "auto"
    is_concurrent_safe = True

    async def execute(self, input: AddInput) -> ToolResult:  # type: ignore[override]
        # 生成 100,000 字符的超大输出
        return ToolResult(call_id="", output="x" * 100_000)


# ---------- 测试：register / get_tool ----------

def test_register_and_get_tool():
    registry = ToolRegistry()
    tool = AddTool()
    registry.register(tool)
    retrieved = registry.get_tool("add")
    assert retrieved is tool


def test_get_nonexistent_tool_returns_none():
    registry = ToolRegistry()
    assert registry.get_tool("nonexistent") is None


def test_register_overwrites_same_name():
    registry = ToolRegistry()
    tool1 = AddTool()
    tool2 = AddTool()
    registry.register(tool1)
    registry.register(tool2)
    assert registry.get_tool("add") is tool2


# ---------- 测试：get_schemas() — deny 不暴露 ----------

def test_get_schemas_excludes_deny_tools():
    registry = ToolRegistry()
    registry.register(AddTool())       # auto
    registry.register(MultiplyTool())  # ask
    registry.register(DangerousTool()) # deny

    schemas = registry.get_schemas()
    names = [s["name"] for s in schemas]

    assert "add" in names
    assert "multiply" in names
    assert "dangerous_op" not in names  # deny 工具不暴露给模型


def test_get_schemas_returns_valid_structure():
    registry = ToolRegistry()
    registry.register(AddTool())
    schemas = registry.get_schemas()
    assert len(schemas) == 1
    schema = schemas[0]
    assert "name" in schema
    assert "description" in schema
    assert "input_schema" in schema


def test_get_schemas_empty_registry():
    registry = ToolRegistry()
    assert registry.get_schemas() == []


# ---------- 测试：execute() — 基础执行 ----------

@pytest.mark.asyncio
async def test_execute_single_tool_call():
    registry = ToolRegistry()
    registry.register(AddTool())

    calls = [ToolCall(id="call_1", name="add", input={"a": 3, "b": 4})]
    results = await registry.execute(calls)

    assert len(results) == 1
    assert results[0].call_id == "call_1"
    assert results[0].output == "7"
    assert results[0].is_error is False


@pytest.mark.asyncio
async def test_execute_fills_call_id_in_result():
    registry = ToolRegistry()
    registry.register(AddTool())

    calls = [ToolCall(id="my_call_id", name="add", input={"a": 1, "b": 2})]
    results = await registry.execute(calls)

    assert results[0].call_id == "my_call_id"


@pytest.mark.asyncio
async def test_execute_unknown_tool_returns_error():
    registry = ToolRegistry()
    calls = [ToolCall(id="call_x", name="nonexistent", input={})]
    results = await registry.execute(calls)
    assert len(results) == 1
    assert results[0].is_error is True
    assert results[0].call_id == "call_x"


@pytest.mark.asyncio
async def test_execute_tool_exception_returns_error():
    registry = ToolRegistry()
    registry.register(ErrorTool())

    calls = [ToolCall(id="err_call", name="error_op", input={"a": 1, "b": 2})]
    results = await registry.execute(calls)

    assert results[0].is_error is True
    assert "tool execution failed" in results[0].output


# ---------- 测试：并发安全分区执行 ----------

@pytest.mark.asyncio
async def test_concurrent_safe_tools_run_in_parallel():
    """两个 concurrent_safe=True 的工具应并行执行（通过 asyncio.gather）"""
    registry = ToolRegistry()
    registry.register(AddTool())      # concurrent_safe=True
    registry.register(MultiplyTool()) # concurrent_safe=True

    calls = [
        ToolCall(id="c1", name="add", input={"a": 1, "b": 2}),
        ToolCall(id="c2", name="multiply", input={"x": 3, "y": 4}),
    ]
    results = await registry.execute(calls)

    result_map = {r.call_id: r for r in results}
    assert result_map["c1"].output == "3"
    assert result_map["c2"].output == "12"


@pytest.mark.asyncio
async def test_serial_tools_preserve_order():
    """concurrent_safe=False 的工具必须串行执行，顺序与 calls 一致"""
    SerialTool.execution_order = []
    registry = ToolRegistry()
    registry.register(SerialTool())

    calls = [
        ToolCall(id="s1", name="serial_op", input={"a": 1, "b": 0}),
        ToolCall(id="s2", name="serial_op", input={"a": 2, "b": 0}),
        ToolCall(id="s3", name="serial_op", input={"a": 3, "b": 0}),
    ]
    results = await registry.execute(calls)

    # 串行执行，顺序必须与 calls 一致
    assert SerialTool.execution_order == [1, 2, 3]
    assert [r.output for r in results] == ["1", "2", "3"]


@pytest.mark.asyncio
async def test_mixed_concurrent_and_serial_tools():
    """混合场景：concurrent_safe 工具并行，serial 工具串行，结果顺序与 calls 保持一致"""
    registry = ToolRegistry()
    registry.register(AddTool())     # concurrent_safe=True
    registry.register(SerialTool())  # concurrent_safe=False

    calls = [
        ToolCall(id="a1", name="add", input={"a": 5, "b": 5}),
        ToolCall(id="s1", name="serial_op", input={"a": 10, "b": 0}),
        ToolCall(id="a2", name="add", input={"a": 20, "b": 20}),
    ]
    results = await registry.execute(calls)

    result_map = {r.call_id: r for r in results}
    assert result_map["a1"].output == "10"
    assert result_map["s1"].output == "10"
    assert result_map["a2"].output == "40"


# ---------- 测试：工具结果截断 ----------

@pytest.mark.asyncio
async def test_large_output_is_truncated():
    """超过 max_result_size 的输出必须被截断"""
    registry = ToolRegistry(max_result_size=50_000)
    registry.register(LargeOutputTool())

    calls = [ToolCall(id="big", name="large_output", input={"a": 1, "b": 2})]
    results = await registry.execute(calls)

    assert len(results[0].output) <= 50_000 + 200  # 截断提示本身有少量文字
    assert "[truncated]" in results[0].output or "截断" in results[0].output


@pytest.mark.asyncio
async def test_output_within_limit_not_truncated():
    registry = ToolRegistry(max_result_size=50_000)
    registry.register(AddTool())

    calls = [ToolCall(id="c1", name="add", input={"a": 1, "b": 1})]
    results = await registry.execute(calls)

    assert results[0].output == "2"


@pytest.mark.asyncio
async def test_custom_max_result_size():
    """可通过构造参数自定义截断阈值"""
    registry = ToolRegistry(max_result_size=10)
    registry.register(AddTool())

    # AddTool 输出 "2"（1字符），不触发截断
    calls = [ToolCall(id="c1", name="add", input={"a": 1, "b": 1})]
    results = await registry.execute(calls)
    assert results[0].output == "2"

    # 用 LargeOutputTool 验证 10 字符阈值确实生效
    registry.register(LargeOutputTool())
    calls = [ToolCall(id="big", name="large_output", input={"a": 1, "b": 2})]
    results = await registry.execute(calls)
    assert len(results[0].output) <= 10 + 200
    assert "[truncated]" in results[0].output or "截断" in results[0].output
```

---

### Step 2 — 跑测试（应全部失败）

```bash
pytest tests/tools/test_registry.py -v 2>&1 | head -40
```

预期：`ModuleNotFoundError: No module named 'neoagent.tools.registry'`

---

### Step 3 — 写实现

文件路径: `neoagent/tools/registry.py`

```python
from __future__ import annotations

import asyncio
from typing import TYPE_CHECKING

from pydantic import BaseModel, ValidationError

from neoagent.core.types import ToolCall, ToolResult

if TYPE_CHECKING:
    from neoagent.tools.base import BaseTool

# 截断提示模板（附在被截断输出的末尾）
_TRUNCATION_NOTICE = (
    "\n\n[truncated: output exceeded {limit} characters. "
    "Showing first {limit} characters only.]"
)


class ToolRegistry:
    """
    工具注册表。

    职责：
    1. 注册 / 查询工具
    2. 向 LLM 暴露 schema（deny 工具不暴露）
    3. 执行工具调用（并发安全分区 + 结果截断）

    设计原则：
    - fail-closed：默认串行，只有明确声明 is_concurrent_safe=True 才并行
    - deny 工具不暴露给模型，保证工具集最小化
    - 工具异常由 Registry 兜底，不向 QueryLoop 泄漏
    - 结果截断防止 context 溢出
    """

    def __init__(self, max_result_size: int = 50_000) -> None:
        """
        Args:
            max_result_size: 单个工具输出的最大字符数。
                             超出部分截断并附加提示。默认 50,000。
        """
        self._tools: dict[str, BaseTool] = {}
        self.max_result_size = max_result_size

    # ------------------------------------------------------------------
    # 注册 & 查询
    # ------------------------------------------------------------------

    def register(self, tool: BaseTool) -> None:
        """注册工具。同名工具后注册覆盖先注册。"""
        self._tools[tool.name] = tool

    def get_tool(self, name: str) -> BaseTool | None:
        """按名称查找工具。不存在返回 None。"""
        return self._tools.get(name)

    # ------------------------------------------------------------------
    # Schema 暴露
    # ------------------------------------------------------------------

    def get_schemas(self) -> list[dict]:
        """
        返回所有非 deny 工具的 API schema。

        deny 工具绝对不暴露给模型（模型甚至不知道它的存在），
        这是最严格的权限控制：连尝试调用的机会都没有。
        """
        return [
            tool.get_schema()
            for tool in self._tools.values()
            if tool.permission != "deny"
        ]

    # ------------------------------------------------------------------
    # 执行
    # ------------------------------------------------------------------

    async def execute(self, calls: list[ToolCall]) -> list[ToolResult]:
        """
        执行工具调用列表，返回结果列表（顺序与 calls 一致）。

        并发安全分区策略（Claude Code 模式，fail-closed）：
        - is_concurrent_safe=True  → asyncio.gather 并行执行
        - is_concurrent_safe=False → 串行逐个执行
        - 混合调用时：先并行执行 safe 组，再串行执行 unsafe 组
        - 最终结果顺序按照原始 calls 顺序重新排列

        每个工具调用的错误单独隔离（一个失败不影响其他调用）。
        """
        # 分区：记录 (原始index, call) 便于最终重排序
        safe_indexed: list[tuple[int, ToolCall]] = []
        unsafe_indexed: list[tuple[int, ToolCall]] = []

        for idx, call in enumerate(calls):
            tool = self._tools.get(call.name)
            if tool is not None and tool.is_concurrent_safe:
                safe_indexed.append((idx, call))
            else:
                unsafe_indexed.append((idx, call))

        # 并行执行 concurrent_safe 工具
        safe_results: list[tuple[int, ToolResult]] = []
        if safe_indexed:
            gathered = await asyncio.gather(
                *[self._run_single(call) for _, call in safe_indexed],
                return_exceptions=False,
            )
            for (idx, _), result in zip(safe_indexed, gathered):
                safe_results.append((idx, result))

        # 串行执行 non-concurrent_safe 工具
        unsafe_results: list[tuple[int, ToolResult]] = []
        for idx, call in unsafe_indexed:
            result = await self._run_single(call)
            unsafe_results.append((idx, result))

        # 按原始 calls 顺序重排结果
        all_results = safe_results + unsafe_results
        all_results.sort(key=lambda x: x[0])
        return [result for _, result in all_results]

    async def _run_single(self, call: ToolCall) -> ToolResult:
        """
        执行单个工具调用，兜底所有异常，填充 call_id，截断过大输出。
        """
        tool = self._tools.get(call.name)

        # 工具不存在
        if tool is None:
            return ToolResult(
                call_id=call.id,
                output=f"Tool '{call.name}' not found in registry.",
                is_error=True,
            )

        # 验证并解析 input
        try:
            parsed_input: BaseModel = tool.input_model.model_validate(call.input)
        except ValidationError as e:
            return ToolResult(
                call_id=call.id,
                output=f"Input validation failed for tool '{call.name}': {e}",
                is_error=True,
            )

        # 执行工具（兜底运行时异常）
        try:
            result = await tool.execute(parsed_input)
        except Exception as e:  # noqa: BLE001
            return ToolResult(
                call_id=call.id,
                output=f"Tool '{call.name}' raised an exception: {e}",
                is_error=True,
            )

        # 填充 call_id（工具实现可以不填）
        result.call_id = call.id

        # 截断过大输出
        result.output = self._truncate(result.output)

        return result

    def _truncate(self, output: str) -> str:
        """
        若 output 超过 max_result_size，截断并附加提示。

        截断提示本身较短（约 80 字符），不计入 max_result_size 限制，
        这是合理的：宁可多出一点，也要让模型知道内容被截断了。
        """
        if len(output) <= self.max_result_size:
            return output
        truncated = output[: self.max_result_size]
        notice = _TRUNCATION_NOTICE.format(limit=self.max_result_size)
        return truncated + notice
```

---

### Step 4 — 跑测试（应全部通过）

```bash
pytest tests/tools/test_registry.py -v
```

预期输出：
```
tests/tools/test_registry.py::test_register_and_get_tool PASSED
tests/tools/test_registry.py::test_get_nonexistent_tool_returns_none PASSED
tests/tools/test_registry.py::test_register_overwrites_same_name PASSED
tests/tools/test_registry.py::test_get_schemas_excludes_deny_tools PASSED
tests/tools/test_registry.py::test_get_schemas_returns_valid_structure PASSED
tests/tools/test_registry.py::test_get_schemas_empty_registry PASSED
tests/tools/test_registry.py::test_execute_single_tool_call PASSED
tests/tools/test_registry.py::test_execute_fills_call_id_in_result PASSED
tests/tools/test_registry.py::test_execute_unknown_tool_returns_error PASSED
tests/tools/test_registry.py::test_execute_tool_exception_returns_error PASSED
tests/tools/test_registry.py::test_concurrent_safe_tools_run_in_parallel PASSED
tests/tools/test_registry.py::test_serial_tools_preserve_order PASSED
tests/tools/test_registry.py::test_mixed_concurrent_and_serial_tools PASSED
tests/tools/test_registry.py::test_large_output_is_truncated PASSED
tests/tools/test_registry.py::test_output_within_limit_not_truncated PASSED
tests/tools/test_registry.py::test_custom_max_result_size PASSED
16 passed in 0.XXs
```

---

### Step 5 — 检查点

- [ ] `register()` 注册，同名覆盖
- [ ] `get_tool()` 查找，不存在返回 `None`
- [ ] `get_schemas()` 不包含 `permission="deny"` 的工具
- [ ] `execute()` 正确填充 `call_id`
- [ ] 未知工具名 → `is_error=True`
- [ ] 工具 `execute()` 抛异常 → `is_error=True`，不向上传播
- [ ] `is_concurrent_safe=True` 的工具用 `asyncio.gather` 并行
- [ ] `is_concurrent_safe=False` 的工具串行，顺序与 calls 一致
- [ ] 结果顺序与 `calls` 顺序一致（混合场景）
- [ ] 超过 `max_result_size` 的输出被截断并附加 `[truncated]` 提示
- [ ] 16 个测试全部通过，0 个跳过

---

## Task 6: Permission Checker (`neoagent/tools/permission.py`)

### Files

- `neoagent/tools/permission.py`
- `tests/tools/test_permission.py`

---

### Step 1 — 写失败测试

文件路径: `tests/tools/test_permission.py`

```python
from __future__ import annotations

from typing import Awaitable, Callable
from unittest.mock import AsyncMock

import pytest
from pydantic import BaseModel

from neoagent.core.types import ToolResult
from neoagent.tools.base import BaseTool
from neoagent.tools.permission import PermissionChecker


# ---------- 测试用 Tool 定义 ----------

class DummyInput(BaseModel):
    value: str = "test"


class AutoTool(BaseTool):
    name = "auto_tool"
    description = "自动允许执行的安全工具"
    input_model = DummyInput
    permission = "auto"
    is_concurrent_safe = True

    async def execute(self, input: DummyInput) -> ToolResult:  # type: ignore[override]
        return ToolResult(call_id="", output="auto")


class DenyTool(BaseTool):
    name = "deny_tool"
    description = "永久拒绝执行的危险工具"
    input_model = DummyInput
    permission = "deny"
    is_concurrent_safe = False

    async def execute(self, input: DummyInput) -> ToolResult:  # type: ignore[override]
        return ToolResult(call_id="", output="deny")


class AskTool(BaseTool):
    name = "ask_tool"
    description = "需要用户确认才能执行的工具"
    input_model = DummyInput
    permission = "ask"
    is_concurrent_safe = False

    async def execute(self, input: DummyInput) -> ToolResult:  # type: ignore[override]
        return ToolResult(call_id="", output="ask")


# ---------- 测试：auto 权限 ----------

@pytest.mark.asyncio
async def test_auto_permission_always_returns_true():
    checker = PermissionChecker()
    tool = AutoTool()
    input_data = DummyInput()
    result = await checker.check(tool, input_data)
    assert result is True


@pytest.mark.asyncio
async def test_auto_permission_with_callback_still_returns_true():
    """auto 权限不调用 callback，直接返回 True"""
    callback = AsyncMock(return_value=False)  # callback 返回 False，但不应被调用
    checker = PermissionChecker(ask_callback=callback)
    tool = AutoTool()
    result = await checker.check(tool, DummyInput())
    assert result is True
    callback.assert_not_called()


# ---------- 测试：deny 权限 ----------

@pytest.mark.asyncio
async def test_deny_permission_always_returns_false():
    checker = PermissionChecker()
    tool = DenyTool()
    result = await checker.check(tool, DummyInput())
    assert result is False


@pytest.mark.asyncio
async def test_deny_permission_with_callback_still_returns_false():
    """deny 权限不调用 callback，直接返回 False"""
    callback = AsyncMock(return_value=True)  # callback 返回 True，但不应被调用
    checker = PermissionChecker(ask_callback=callback)
    tool = DenyTool()
    result = await checker.check(tool, DummyInput())
    assert result is False
    callback.assert_not_called()


# ---------- 测试：ask 权限 — 无 callback ----------

@pytest.mark.asyncio
async def test_ask_permission_without_callback_defaults_to_true():
    """ask 模式无 callback 时，默认允许（宽松默认，适合开发环境）"""
    checker = PermissionChecker()
    tool = AskTool()
    result = await checker.check(tool, DummyInput())
    assert result is True


# ---------- 测试：ask 权限 — 有 callback ----------

@pytest.mark.asyncio
async def test_ask_permission_callback_returns_true():
    callback = AsyncMock(return_value=True)
    checker = PermissionChecker(ask_callback=callback)
    tool = AskTool()
    input_data = DummyInput(value="hello")
    result = await checker.check(tool, input_data)
    assert result is True
    callback.assert_called_once()


@pytest.mark.asyncio
async def test_ask_permission_callback_returns_false():
    callback = AsyncMock(return_value=False)
    checker = PermissionChecker(ask_callback=callback)
    tool = AskTool()
    result = await checker.check(tool, DummyInput())
    assert result is False
    callback.assert_called_once()


@pytest.mark.asyncio
async def test_ask_permission_callback_receives_correct_args():
    """
    callback 应收到 (tool_name: str, tool_description: str, input_dict: dict)
    让 callback 实现者可以展示足够信息给用户确认
    """
    received_args: list = []

    async def capture_callback(name: str, description: str, input_dict: dict) -> bool:
        received_args.extend([name, description, input_dict])
        return True

    checker = PermissionChecker(ask_callback=capture_callback)
    tool = AskTool()
    input_data = DummyInput(value="sensitive_value")
    await checker.check(tool, input_data)

    assert received_args[0] == "ask_tool"
    assert received_args[1] == "需要用户确认才能执行的工具"
    assert received_args[2] == {"value": "sensitive_value"}


@pytest.mark.asyncio
async def test_ask_permission_callback_called_each_time():
    """每次 check 都调用一次 callback（不缓存结果）"""
    callback = AsyncMock(return_value=True)
    checker = PermissionChecker(ask_callback=callback)
    tool = AskTool()

    await checker.check(tool, DummyInput())
    await checker.check(tool, DummyInput())
    await checker.check(tool, DummyInput())

    assert callback.call_count == 3


# ---------- 测试：多种工具混合 ----------

@pytest.mark.asyncio
async def test_mixed_permissions_with_callback():
    """同一个 checker 处理三种权限的工具"""
    callback = AsyncMock(return_value=True)
    checker = PermissionChecker(ask_callback=callback)

    assert await checker.check(AutoTool(), DummyInput()) is True
    assert await checker.check(DenyTool(), DummyInput()) is False
    assert await checker.check(AskTool(), DummyInput()) is True

    # callback 只被 ask 工具触发一次
    assert callback.call_count == 1


@pytest.mark.asyncio
async def test_checker_is_stateless_between_calls():
    """PermissionChecker 自身无状态，结果完全由 tool.permission 和 callback 决定"""
    callback_results = [True, False, True]
    call_count = 0

    async def varying_callback(name: str, desc: str, inp: dict) -> bool:
        nonlocal call_count
        result = callback_results[call_count % len(callback_results)]
        call_count += 1
        return result

    checker = PermissionChecker(ask_callback=varying_callback)
    tool = AskTool()

    results = []
    for _ in range(3):
        results.append(await checker.check(tool, DummyInput()))

    assert results == [True, False, True]
```

---

### Step 2 — 跑测试（应全部失败）

```bash
pytest tests/tools/test_permission.py -v 2>&1 | head -40
```

预期：`ModuleNotFoundError: No module named 'neoagent.tools.permission'`

---

### Step 3 — 写实现

文件路径: `neoagent/tools/permission.py`

```python
from __future__ import annotations

from typing import Awaitable, Callable

from pydantic import BaseModel

from neoagent.tools.base import BaseTool

# callback 签名：(tool_name, tool_description, input_as_dict) -> bool
AskCallback = Callable[[str, str, dict], Awaitable[bool]]


class PermissionChecker:
    """
    工具权限检查器。

    三级权限模型（来自 Claude Code，KB Tool System 推荐）：

    - "auto" → 无需确认，直接允许
      适用：只读工具（Grep、Glob、Read）、无副作用的计算工具
      原则：工具作者声明安全，框架信任

    - "deny" → 永久拒绝，不可绕过
      适用：危险操作（不可逆删除、敏感数据访问）
      原则：连 callback 都不调，从协议层彻底屏蔽

    - "ask" → 调用 ask_callback 由外部决策
      适用：有副作用但可控的工具（Write、Edit、Bash）
      原则：运行时动态决策，支持 CLI 交互 / CI 自动允许 / 安全审计

    无 callback 时 ask 默认 True：
      开发调试阶段友好，生产部署时必须传入 callback 做审计。

    设计约束：
    - PermissionChecker 自身无状态（无缓存、无记忆）
    - 结果完全由 tool.permission + callback 决定
    - 每次调用都是独立判断（便于 callback 实现细粒度审计）
    """

    def __init__(
        self,
        ask_callback: AskCallback | None = None,
    ) -> None:
        """
        Args:
            ask_callback: 可选的异步 callback，签名为
                          (tool_name: str, tool_description: str, input_dict: dict) -> bool
                          用于在 "ask" 权限模式下向用户或审计系统请求确认。
                          None 时默认允许（适合开发环境）。
        """
        self._ask_callback = ask_callback

    async def check(self, tool: BaseTool, input: BaseModel) -> bool:
        """
        检查是否允许执行该工具。

        Args:
            tool: 要执行的工具实例
            input: 已验证的 Pydantic input 实例（用于传给 callback 展示）

        Returns:
            True  → 允许执行
            False → 拒绝执行
        """
        if tool.permission == "auto":
            return True

        if tool.permission == "deny":
            return False

        # permission == "ask"
        return await self._handle_ask(tool, input)

    async def _handle_ask(self, tool: BaseTool, input: BaseModel) -> bool:
        """
        处理 ask 权限：
        - 有 callback → 调用 callback(name, description, input_dict)
        - 无 callback → 默认 True（开发友好，生产应配置 callback）
        """
        if self._ask_callback is None:
            return True

        input_dict = input.model_dump()
        return await self._ask_callback(tool.name, tool.description, input_dict)
```

---

### Step 4 — 跑测试（应全部通过）

```bash
pytest tests/tools/test_permission.py -v
```

预期输出：
```
tests/tools/test_permission.py::test_auto_permission_always_returns_true PASSED
tests/tools/test_permission.py::test_auto_permission_with_callback_still_returns_true PASSED
tests/tools/test_permission.py::test_deny_permission_always_returns_false PASSED
tests/tools/test_permission.py::test_deny_permission_with_callback_still_returns_false PASSED
tests/tools/test_permission.py::test_ask_permission_without_callback_defaults_to_true PASSED
tests/tools/test_permission.py::test_ask_permission_callback_returns_true PASSED
tests/tools/test_permission.py::test_ask_permission_callback_returns_false PASSED
tests/tools/test_permission.py::test_ask_permission_callback_receives_correct_args PASSED
tests/tools/test_permission.py::test_ask_permission_callback_called_each_time PASSED
tests/tools/test_permission.py::test_mixed_permissions_with_callback PASSED
tests/tools/test_permission.py::test_checker_is_stateless_between_calls PASSED
11 passed in 0.XXs
```

---

### Step 5 — 检查点

- [ ] `auto` → 直接返回 `True`，不调用 callback
- [ ] `deny` → 直接返回 `False`，不调用 callback
- [ ] `ask` + 无 callback → 返回 `True`（开发默认宽松）
- [ ] `ask` + callback 返回 `True` → 允许
- [ ] `ask` + callback 返回 `False` → 拒绝
- [ ] callback 参数正确：`(tool_name: str, tool_description: str, input_dict: dict)`
- [ ] `input_dict` 来自 `input.model_dump()`（Pydantic v2 风格）
- [ ] 每次 `check()` 调用 callback 一次（无缓存）
- [ ] `PermissionChecker` 自身无状态
- [ ] 11 个测试全部通过，0 个跳过

---

## Part 2 整体验证

完成 Task 4、5、6 后，运行完整测试套件确认无回归：

```bash
pytest tests/tools/ -v --tb=short
```

预期：37 个测试全部通过（Task 4: 10 + Task 5: 16 + Task 6: 11）

检查模块导入链正常：

```python
from neoagent.tools.base import BaseTool
from neoagent.tools.registry import ToolRegistry
from neoagent.tools.permission import PermissionChecker
```

---

## 关键设计决策备忘

| 决策点 | 选择 | 依据 |
|--------|------|------|
| `input_model` Pydantic 一套两用 | 验证 + schema 生成共用同一个 BaseModel | OpenHarness 模式，减少定义冗余 |
| schema 格式 | `{name, description, input_schema}` | Anthropic API 约定格式 |
| 并发默认串行 | `is_concurrent_safe=False` | Claude Code fail-closed 策略，安全优先 |
| deny 工具不暴露 schema | `get_schemas()` 过滤 deny | 最严格权限控制，模型甚至不知道工具存在 |
| 结果截断默认 50,000 字符 | 可通过构造参数覆盖 | Claude Code + Hermes 防 context 溢出经验 |
| ask 无 callback 默认 True | 开发友好，生产必须配置 | 降低上手门槛，不强制用户实现 callback |
| callback 签名 `(name, desc, input_dict)` | 三参数足够 UI 展示 | 最小接口原则，不传整个 tool 对象 |
| PermissionChecker 无状态 | 每次独立判断 | 支持 callback 实现细粒度审计策略 |
