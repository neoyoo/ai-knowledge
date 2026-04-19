# neoagent v1 Implementation Plan — Part 2 (Tasks 4–6)

> **For agentic workers:** Use superpowers:subagent-driven-development to execute task-by-task. Each task has a TDD loop: write failing test → verify failure → write implementation → verify passing → commit.

**Goal:** Implement the Tool System foundation — BaseTool ABC, ToolRegistry, and PermissionChecker — with full test coverage using pytest-asyncio.

**Spec:** `docs/superpowers/specs/2026-04-10-neoagent-design.md` (§8 Tool System)

**Part 1 prerequisite:** `neoagent/core/types.py` must exist with `Message`, `ToolCall`, `ToolResult` dataclasses.

**Python requirements:** Python 3.11+, `from __future__ import annotations`, async/await, pydantic v2, pytest-asyncio

---

## File Map

### Create:
- `neoagent/tools/__init__.py`
- `neoagent/tools/base.py`
- `neoagent/tools/registry.py`
- `neoagent/tools/permission.py`
- `tests/tools/__init__.py`
- `tests/tools/test_base.py`
- `tests/tools/test_registry.py`
- `tests/tools/test_permission.py`

---

## Task 4: Tool 基类

**Files:**
- Create: `neoagent/tools/__init__.py`
- Create: `neoagent/tools/base.py`
- Create: `tests/tools/__init__.py`
- Create: `tests/tools/test_base.py`

---

### Step 4.1: 创建 `tests/tools/__init__.py` 和 `neoagent/tools/__init__.py`

```bash
mkdir -p neoagent/tools tests/tools
touch neoagent/tools/__init__.py tests/tools/__init__.py
```

验证：
```bash
ls neoagent/tools/__init__.py tests/tools/__init__.py
```

Expected: 两个文件存在，无报错。

---

### Step 4.2: 写失败测试 `tests/tools/test_base.py`

- [ ] **写入以下内容到 `tests/tools/test_base.py`：**

```python
from __future__ import annotations

import pytest
from pydantic import BaseModel

from neoagent.tools.base import BaseTool
from neoagent.core.types import ToolResult


class AddInput(BaseModel):
    a: int
    b: int


class AddTool(BaseTool):
    name: str = "add"
    description: str = "将两个整数相加并返回结果"
    input_model: type[BaseModel] = AddInput
    permission: str = "auto"
    is_concurrent_safe: bool = True

    async def execute(self, input: BaseModel) -> ToolResult:
        assert isinstance(input, AddInput)
        result = input.a + input.b
        return ToolResult(call_id="test-id", output=str(result))


class TestBaseTool:
    def test_get_schema_has_required_keys(self):
        """get_schema() 必须包含 name、description、input_schema 三个顶层键"""
        tool = AddTool()
        schema = tool.get_schema()
        assert "name" in schema
        assert "description" in schema
        assert "input_schema" in schema

    def test_get_schema_name_matches_tool_name(self):
        """schema['name'] 必须等于 tool.name"""
        tool = AddTool()
        schema = tool.get_schema()
        assert schema["name"] == "add"

    def test_get_schema_description_matches(self):
        """schema['description'] 必须等于 tool.description"""
        tool = AddTool()
        schema = tool.get_schema()
        assert schema["description"] == "将两个整数相加并返回结果"

    def test_get_schema_input_schema_has_properties(self):
        """input_schema 必须包含 a 和 b 两个 properties"""
        tool = AddTool()
        schema = tool.get_schema()
        props = schema["input_schema"]["properties"]
        assert "a" in props
        assert "b" in props

    def test_get_schema_input_schema_type_is_object(self):
        """input_schema['type'] 必须是 'object'"""
        tool = AddTool()
        schema = tool.get_schema()
        assert schema["input_schema"]["type"] == "object"

    def test_permission_default_is_ask(self):
        """permission 默认值必须是 'ask'"""
        class MinimalInput(BaseModel):
            x: str

        class MinimalTool(BaseTool):
            name: str = "minimal"
            description: str = "最简工具用于测试默认权限"
            input_model: type[BaseModel] = MinimalInput

            async def execute(self, input: BaseModel) -> ToolResult:
                return ToolResult(call_id="x", output="ok")

        tool = MinimalTool()
        assert tool.permission == "ask"

    def test_is_concurrent_safe_default_is_false(self):
        """is_concurrent_safe 默认值必须是 False"""
        class MinimalInput(BaseModel):
            x: str

        class MinimalTool(BaseTool):
            name: str = "minimal2"
            description: str = "最简工具用于测试默认并发安全标志"
            input_model: type[BaseModel] = MinimalInput

            async def execute(self, input: BaseModel) -> ToolResult:
                return ToolResult(call_id="x", output="ok")

        tool = MinimalTool()
        assert tool.is_concurrent_safe is False

    @pytest.mark.asyncio
    async def test_execute_returns_tool_result(self):
        """execute() 必须返回 ToolResult 实例"""
        tool = AddTool()
        input_data = AddInput(a=3, b=4)
        result = await tool.execute(input_data)
        assert isinstance(result, ToolResult)
        assert result.output == "7"
        assert result.is_error is False
```

---

### Step 4.3: 运行测试，确认失败

```bash
cd neoagent && python -m pytest tests/tools/test_base.py -v 2>&1 | head -40
```

Expected: `ModuleNotFoundError` 或 `ImportError`（`neoagent.tools.base` 不存在）。

---

### Step 4.4: 写实现 `neoagent/tools/base.py`

- [ ] **写入以下内容到 `neoagent/tools/base.py`：**

```python
from __future__ import annotations

from abc import ABC, abstractmethod
from typing import Literal

from pydantic import BaseModel

from neoagent.core.types import ToolResult


class BaseTool(ABC):
    """所有工具的抽象基类。

    子类必须：
    - 定义 name、description、input_model 类属性
    - 实现 execute(input) 异步方法
    """

    name: str
    description: str  # 20-50 字，KB: 超 200 字浪费 token
    input_model: type[BaseModel]
    permission: Literal["auto", "ask", "deny"] = "ask"  # Claude Code: 三级权限
    is_concurrent_safe: bool = False  # Claude Code: fail-closed 默认串行

    @abstractmethod
    async def execute(self, input: BaseModel) -> ToolResult:
        """执行工具逻辑，返回 ToolResult。

        Args:
            input: 已通过 input_model 验证的 Pydantic 实例

        Returns:
            ToolResult，包含 output 字符串和 is_error 标志
        """
        ...

    def get_schema(self) -> dict:
        """从 input_model 自动生成兼容 Anthropic tool API 的 JSON Schema。

        格式：
        {
            "name": "tool_name",
            "description": "tool description",
            "input_schema": { ...pydantic json schema... }
        }
        """
        raw_schema = self.input_model.model_json_schema()

        # 移除 Pydantic 自动加的 title 字段（Anthropic API 不需要）
        raw_schema.pop("title", None)

        return {
            "name": self.name,
            "description": self.description,
            "input_schema": raw_schema,
        }
```

---

### Step 4.5: 运行测试，确认全部通过

```bash
cd neoagent && python -m pytest tests/tools/test_base.py -v
```

Expected：所有测试通过，`8 passed`。

---

### Step 4.6: 提交

```bash
cd neoagent && git add neoagent/tools/__init__.py neoagent/tools/base.py tests/tools/__init__.py tests/tools/test_base.py
git commit -m "feat(tools): add BaseTool ABC with get_schema() and three-level permission"
```

---

## Task 5: Tool Registry

**Files:**
- Create: `neoagent/tools/registry.py`
- Create: `tests/tools/test_registry.py`

---

### Step 5.1: 写失败测试 `tests/tools/test_registry.py`

- [ ] **写入以下内容到 `tests/tools/test_registry.py`：**

```python
from __future__ import annotations

import asyncio
import pytest
from pydantic import BaseModel

from neoagent.tools.base import BaseTool
from neoagent.tools.registry import ToolRegistry
from neoagent.core.types import ToolCall, ToolResult


# ── 测试用工具 ──────────────────────────────────────────────

class AddInput(BaseModel):
    a: int
    b: int


class AddTool(BaseTool):
    name: str = "add"
    description: str = "将两个整数相加并返回结果"
    input_model: type[BaseModel] = AddInput
    permission: str = "auto"
    is_concurrent_safe: bool = True

    async def execute(self, input: BaseModel) -> ToolResult:
        assert isinstance(input, AddInput)
        return ToolResult(call_id="", output=str(input.a + input.b))


class EchoInput(BaseModel):
    message: str


class EchoTool(BaseTool):
    name: str = "echo"
    description: str = "回显输入的消息字符串"
    input_model: type[BaseModel] = EchoInput
    permission: str = "ask"
    is_concurrent_safe: bool = False

    async def execute(self, input: BaseModel) -> ToolResult:
        assert isinstance(input, EchoInput)
        return ToolResult(call_id="", output=input.message)


class DenyInput(BaseModel):
    cmd: str


class DenyTool(BaseTool):
    name: str = "secret"
    description: str = "被拒绝的工具，不应暴露给模型"
    input_model: type[BaseModel] = DenyInput
    permission: str = "deny"
    is_concurrent_safe: bool = False

    async def execute(self, input: BaseModel) -> ToolResult:
        return ToolResult(call_id="", output="should never run")


class SlowInput(BaseModel):
    delay: float
    value: str


_execution_order: list[str] = []


class SlowSerialTool(BaseTool):
    name: str = "slow_serial"
    description: str = "模拟慢速串行工具用于测试执行顺序"
    input_model: type[BaseModel] = SlowInput
    permission: str = "auto"
    is_concurrent_safe: bool = False

    async def execute(self, input: BaseModel) -> ToolResult:
        assert isinstance(input, SlowInput)
        await asyncio.sleep(input.delay)
        _execution_order.append(input.value)
        return ToolResult(call_id="", output=input.value)


class SlowConcurrentTool(BaseTool):
    name: str = "slow_concurrent"
    description: str = "模拟慢速并发工具用于测试并行执行"
    input_model: type[BaseModel] = SlowInput
    permission: str = "auto"
    is_concurrent_safe: bool = True

    async def execute(self, input: BaseModel) -> ToolResult:
        assert isinstance(input, SlowInput)
        await asyncio.sleep(input.delay)
        return ToolResult(call_id="", output=input.value)


# ── 测试 ─────────────────────────────────────────────────────

class TestToolRegistryRegister:
    def test_register_and_get_tool(self):
        """register() 后 get_tool() 能取回同一工具"""
        registry = ToolRegistry()
        tool = AddTool()
        registry.register(tool)
        assert registry.get_tool("add") is tool

    def test_get_tool_returns_none_for_unknown(self):
        """get_tool() 对未知名称返回 None"""
        registry = ToolRegistry()
        assert registry.get_tool("nonexistent") is None

    def test_register_duplicate_raises(self):
        """注册同名工具必须抛出 ValueError"""
        registry = ToolRegistry()
        registry.register(AddTool())
        with pytest.raises(ValueError, match="already registered"):
            registry.register(AddTool())

    def test_register_multiple_tools(self):
        """注册多个不同名称的工具，均可取回"""
        registry = ToolRegistry()
        registry.register(AddTool())
        registry.register(EchoTool())
        assert registry.get_tool("add") is not None
        assert registry.get_tool("echo") is not None


class TestToolRegistryGetSchemas:
    def test_get_schemas_excludes_deny(self):
        """get_schemas() 不返回 permission='deny' 的工具"""
        registry = ToolRegistry()
        registry.register(AddTool())
        registry.register(DenyTool())
        schemas = registry.get_schemas()
        names = [s["name"] for s in schemas]
        assert "add" in names
        assert "secret" not in names

    def test_get_schemas_includes_auto_and_ask(self):
        """get_schemas() 返回 auto 和 ask 权限的工具"""
        registry = ToolRegistry()
        registry.register(AddTool())    # auto
        registry.register(EchoTool())  # ask
        registry.register(DenyTool())  # deny
        schemas = registry.get_schemas()
        names = [s["name"] for s in schemas]
        assert "add" in names
        assert "echo" in names
        assert len(names) == 2

    def test_get_schemas_returns_valid_anthropic_format(self):
        """每个 schema 必须包含 name、description、input_schema"""
        registry = ToolRegistry()
        registry.register(AddTool())
        schemas = registry.get_schemas()
        assert len(schemas) == 1
        schema = schemas[0]
        assert "name" in schema
        assert "description" in schema
        assert "input_schema" in schema


class TestToolRegistryExecute:
    @pytest.mark.asyncio
    async def test_execute_single_tool(self):
        """执行单个 ToolCall，返回正确的 ToolResult"""
        registry = ToolRegistry()
        registry.register(AddTool())
        calls = [ToolCall(id="c1", name="add", input={"a": 3, "b": 4})]
        results = await registry.execute(calls)
        assert len(results) == 1
        assert results[0].output == "7"
        assert results[0].call_id == "c1"
        assert results[0].is_error is False

    @pytest.mark.asyncio
    async def test_execute_unknown_tool_returns_error(self):
        """调用未注册的工具，返回 is_error=True 的 ToolResult"""
        registry = ToolRegistry()
        calls = [ToolCall(id="c1", name="unknown_tool", input={})]
        results = await registry.execute(calls)
        assert len(results) == 1
        assert results[0].is_error is True
        assert results[0].call_id == "c1"
        assert "unknown_tool" in results[0].output

    @pytest.mark.asyncio
    async def test_execute_concurrent_safe_runs_in_parallel(self):
        """is_concurrent_safe=True 的工具应并行执行，总时间近似最长单个任务"""
        registry = ToolRegistry()
        registry.register(SlowConcurrentTool())

        calls = [
            ToolCall(id="c1", name="slow_concurrent", input={"delay": 0.1, "value": "a"}),
            ToolCall(id="c2", name="slow_concurrent", input={"delay": 0.1, "value": "b"}),
            ToolCall(id="c3", name="slow_concurrent", input={"delay": 0.1, "value": "c"}),
        ]

        import time
        start = time.monotonic()
        results = await registry.execute(calls)
        elapsed = time.monotonic() - start

        assert len(results) == 3
        # 并行执行，3 个各 0.1s 的任务应在 0.25s 内完成（串行需 0.3s）
        assert elapsed < 0.25, f"Expected parallel execution, got {elapsed:.3f}s"

    @pytest.mark.asyncio
    async def test_execute_unsafe_runs_serially(self):
        """is_concurrent_safe=False 的工具应串行执行"""
        global _execution_order
        _execution_order = []

        registry = ToolRegistry()
        registry.register(SlowSerialTool())

        calls = [
            ToolCall(id="c1", name="slow_serial", input={"delay": 0.05, "value": "first"}),
            ToolCall(id="c2", name="slow_serial", input={"delay": 0.05, "value": "second"}),
            ToolCall(id="c3", name="slow_serial", input={"delay": 0.05, "value": "third"}),
        ]

        results = await registry.execute(calls)
        assert len(results) == 3
        # 串行：执行顺序必须确定
        assert _execution_order == ["first", "second", "third"]

    @pytest.mark.asyncio
    async def test_execute_result_truncation(self):
        """output 超过 max_result_size 时截断并标注"""
        registry = ToolRegistry(max_result_size=20)
        registry.register(EchoTool())

        long_message = "x" * 100
        calls = [ToolCall(id="c1", name="echo", input={"message": long_message})]
        results = await registry.execute(calls)

        assert len(results) == 1
        output = results[0].output
        assert len(output) <= 100  # 必须被截断
        assert "[truncated]" in output

    @pytest.mark.asyncio
    async def test_execute_empty_calls_returns_empty(self):
        """空调用列表返回空结果列表"""
        registry = ToolRegistry()
        results = await registry.execute([])
        assert results == []

    @pytest.mark.asyncio
    async def test_execute_mixed_concurrent_and_serial(self):
        """混合并发安全和不安全的工具：并发安全的并行，不安全的串行"""
        registry = ToolRegistry()
        registry.register(AddTool())    # concurrent_safe=True
        registry.register(EchoTool())  # concurrent_safe=False

        calls = [
            ToolCall(id="c1", name="add", input={"a": 1, "b": 2}),
            ToolCall(id="c2", name="echo", input={"message": "hello"}),
            ToolCall(id="c3", name="add", input={"a": 10, "b": 20}),
        ]
        results = await registry.execute(calls)
        assert len(results) == 3
        # 结果与调用顺序对应
        result_map = {r.call_id: r for r in results}
        assert result_map["c1"].output == "3"
        assert result_map["c2"].output == "hello"
        assert result_map["c3"].output == "30"
```

---

### Step 5.2: 运行测试，确认失败

```bash
cd neoagent && python -m pytest tests/tools/test_registry.py -v 2>&1 | head -40
```

Expected: `ModuleNotFoundError`（`neoagent.tools.registry` 不存在）。

---

### Step 5.3: 写实现 `neoagent/tools/registry.py`

- [ ] **写入以下内容到 `neoagent/tools/registry.py`：**

```python
from __future__ import annotations

import asyncio
from typing import Sequence

from pydantic import BaseModel

from neoagent.core.types import ToolCall, ToolResult
from neoagent.tools.base import BaseTool

_TRUNCATION_SUFFIX = "... [truncated]"
_DEFAULT_MAX_RESULT_SIZE = 50_000  # 字符数，参考 Claude Code + Hermes 设计


class ToolRegistry:
    """工具注册表 — 静态注册，不支持运行时动态增删（KB 方案 A）。

    设计决策：
    - deny 权限工具注册后不暴露给模型，也不允许执行（执行直接报错）
    - 并发安全判定：fail-closed，默认串行（Claude Code 设计）
    - 结果截断：output 超过 max_result_size 截断并标注，防 context 溢出
    """

    def __init__(self, max_result_size: int = _DEFAULT_MAX_RESULT_SIZE) -> None:
        self._tools: dict[str, BaseTool] = {}
        self.max_result_size = max_result_size

    # ── 注册 ──────────────────────────────────────────────────

    def register(self, tool: BaseTool) -> None:
        """注册工具。同名工具重复注册抛出 ValueError。"""
        if tool.name in self._tools:
            raise ValueError(
                f"Tool '{tool.name}' is already registered. "
                f"Use a different name or remove the existing tool first."
            )
        self._tools[tool.name] = tool

    def get_tool(self, name: str) -> BaseTool | None:
        """按名称查找工具，不存在返回 None。"""
        return self._tools.get(name)

    # ── Schema 暴露 ───────────────────────────────────────────

    def get_schemas(self) -> list[dict]:
        """返回所有非 deny 工具的 JSON Schema（Anthropic API 格式）。

        deny 权限工具不暴露给模型——模型看不到就不会调用。
        """
        return [
            tool.get_schema()
            for tool in self._tools.values()
            if tool.permission != "deny"
        ]

    # ── 执行 ─────────────────────────────────────────────────

    async def execute(self, calls: Sequence[ToolCall]) -> list[ToolResult]:
        """执行一批工具调用，返回对应顺序的 ToolResult 列表。

        执行策略：
        - is_concurrent_safe=True 的调用用 asyncio.gather() 并行
        - is_concurrent_safe=False 的调用逐个串行
        - 最终结果按原始调用顺序排列
        """
        if not calls:
            return []

        safe_calls, unsafe_calls = self._partition_by_concurrency(list(calls))

        # 并行执行安全工具
        safe_results = await asyncio.gather(
            *[self._run_single(call) for call in safe_calls]
        )

        # 串行执行不安全工具
        unsafe_results: list[ToolResult] = []
        for call in unsafe_calls:
            result = await self._run_single(call)
            unsafe_results.append(result)

        # 按原始调用顺序重组结果
        all_results = list(safe_results) + unsafe_results
        result_map = {r.call_id: r for r in all_results}
        return [result_map[call.id] for call in calls]

    def _partition_by_concurrency(
        self, calls: list[ToolCall]
    ) -> tuple[list[ToolCall], list[ToolCall]]:
        """将调用按并发安全性分成两组。

        未注册工具归入不安全组（串行处理，执行时返回错误）。
        """
        safe: list[ToolCall] = []
        unsafe: list[ToolCall] = []
        for call in calls:
            tool = self._tools.get(call.name)
            if tool is not None and tool.is_concurrent_safe:
                safe.append(call)
            else:
                unsafe.append(call)
        return safe, unsafe

    async def _run_single(self, call: ToolCall) -> ToolResult:
        """执行单个工具调用，处理工具不存在和执行异常两种错误。"""
        tool = self._tools.get(call.name)

        if tool is None:
            return ToolResult(
                call_id=call.id,
                output=f"Error: Tool '{call.name}' is not registered.",
                is_error=True,
            )

        try:
            # 用 input_model 验证并转换 input dict
            validated_input: BaseModel = tool.input_model.model_validate(call.input)
            result = await tool.execute(validated_input)
            # 修正 call_id（工具实现可能不知道调用 id）
            result = ToolResult(
                call_id=call.id,
                output=result.output,
                is_error=result.is_error,
            )
        except Exception as exc:
            result = ToolResult(
                call_id=call.id,
                output=f"Error executing tool '{call.name}': {exc}",
                is_error=True,
            )

        return self._truncate_result(result)

    def _truncate_result(self, result: ToolResult) -> ToolResult:
        """若 output 超过 max_result_size，截断并追加标注。"""
        if len(result.output) <= self.max_result_size:
            return result

        truncated_output = (
            result.output[: self.max_result_size - len(_TRUNCATION_SUFFIX)]
            + _TRUNCATION_SUFFIX
        )
        return ToolResult(
            call_id=result.call_id,
            output=truncated_output,
            is_error=result.is_error,
        )
```

---

### Step 5.4: 运行测试，确认全部通过

```bash
cd neoagent && python -m pytest tests/tools/test_registry.py -v
```

Expected：所有测试通过，`11 passed`。

---

### Step 5.5: 提交

```bash
cd neoagent && git add neoagent/tools/registry.py tests/tools/test_registry.py
git commit -m "feat(tools): add ToolRegistry with concurrent partitioning and result truncation"
```

---

## Task 6: Permission Checker

**Files:**
- Create: `neoagent/tools/permission.py`
- Create: `tests/tools/test_permission.py`

---

### Step 6.1: 写失败测试 `tests/tools/test_permission.py`

- [ ] **写入以下内容到 `tests/tools/test_permission.py`：**

```python
from __future__ import annotations

import pytest
from pydantic import BaseModel

from neoagent.tools.base import BaseTool
from neoagent.tools.permission import PermissionChecker
from neoagent.core.types import ToolResult


# ── 测试用工具 ──────────────────────────────────────────────

class DummyInput(BaseModel):
    value: str


def _make_tool(permission: str, name: str = "dummy") -> BaseTool:
    class _Tool(BaseTool):
        name: str = name
        description: str = f"测试工具，权限为 {permission}"
        input_model: type[BaseModel] = DummyInput
        is_concurrent_safe: bool = False

        async def execute(self, input: BaseModel) -> ToolResult:
            return ToolResult(call_id="", output="ok")

    _Tool.permission = permission  # type: ignore[attr-defined]
    return _Tool()


# ── 测试 ─────────────────────────────────────────────────────

class TestPermissionCheckerAuto:
    @pytest.mark.asyncio
    async def test_auto_permission_always_true(self):
        """permission='auto' 无条件返回 True"""
        checker = PermissionChecker()
        tool = _make_tool("auto")
        result = await checker.check(tool, DummyInput(value="test"))
        assert result is True

    @pytest.mark.asyncio
    async def test_auto_permission_ignores_callback(self):
        """permission='auto' 时，即使有 ask_callback 也不调用"""
        callback_called = False

        async def callback(tool_name: str, description: str, input_data: dict) -> bool:
            nonlocal callback_called
            callback_called = True
            return False  # 如果被调用，返回 False

        checker = PermissionChecker(ask_callback=callback)
        tool = _make_tool("auto")
        result = await checker.check(tool, DummyInput(value="test"))

        assert result is True
        assert callback_called is False, "auto 权限不应触发 callback"


class TestPermissionCheckerDeny:
    @pytest.mark.asyncio
    async def test_deny_permission_always_false(self):
        """permission='deny' 无条件返回 False"""
        checker = PermissionChecker()
        tool = _make_tool("deny")
        result = await checker.check(tool, DummyInput(value="test"))
        assert result is False

    @pytest.mark.asyncio
    async def test_deny_permission_ignores_callback(self):
        """permission='deny' 时，即使有 ask_callback 也不调用"""
        callback_called = False

        async def callback(tool_name: str, description: str, input_data: dict) -> bool:
            nonlocal callback_called
            callback_called = True
            return True  # 如果被调用，返回 True

        checker = PermissionChecker(ask_callback=callback)
        tool = _make_tool("deny")
        result = await checker.check(tool, DummyInput(value="test"))

        assert result is False
        assert callback_called is False, "deny 权限不应触发 callback"


class TestPermissionCheckerAsk:
    @pytest.mark.asyncio
    async def test_ask_without_callback_defaults_true(self):
        """permission='ask' 且 ask_callback=None 时，默认返回 True"""
        checker = PermissionChecker(ask_callback=None)
        tool = _make_tool("ask")
        result = await checker.check(tool, DummyInput(value="test"))
        assert result is True

    @pytest.mark.asyncio
    async def test_ask_with_callback_returning_true(self):
        """permission='ask' 且 callback 返回 True 时，check 返回 True"""
        async def callback(tool_name: str, description: str, input_data: dict) -> bool:
            return True

        checker = PermissionChecker(ask_callback=callback)
        tool = _make_tool("ask")
        result = await checker.check(tool, DummyInput(value="test"))
        assert result is True

    @pytest.mark.asyncio
    async def test_ask_with_callback_returning_false(self):
        """permission='ask' 且 callback 返回 False 时，check 返回 False"""
        async def callback(tool_name: str, description: str, input_data: dict) -> bool:
            return False

        checker = PermissionChecker(ask_callback=callback)
        tool = _make_tool("ask")
        result = await checker.check(tool, DummyInput(value="test"))
        assert result is False

    @pytest.mark.asyncio
    async def test_ask_callback_receives_correct_args(self):
        """callback 必须接收到 tool_name、description、input_data 三个参数"""
        received_args: dict = {}

        async def callback(tool_name: str, description: str, input_data: dict) -> bool:
            received_args["tool_name"] = tool_name
            received_args["description"] = description
            received_args["input_data"] = input_data
            return True

        checker = PermissionChecker(ask_callback=callback)
        tool = _make_tool("ask", name="my_tool")
        input_data = DummyInput(value="hello")
        await checker.check(tool, input_data)

        assert received_args["tool_name"] == "my_tool"
        assert isinstance(received_args["description"], str)
        assert received_args["input_data"] == {"value": "hello"}

    @pytest.mark.asyncio
    async def test_ask_callback_can_be_stateful(self):
        """callback 支持有状态的逻辑（例如记录调用次数）"""
        call_count = 0

        async def callback(tool_name: str, description: str, input_data: dict) -> bool:
            nonlocal call_count
            call_count += 1
            return call_count <= 1  # 第一次 True，之后 False

        checker = PermissionChecker(ask_callback=callback)
        tool = _make_tool("ask")
        dummy_input = DummyInput(value="x")

        first = await checker.check(tool, dummy_input)
        second = await checker.check(tool, dummy_input)

        assert first is True
        assert second is False
        assert call_count == 2
```

---

### Step 6.2: 运行测试，确认失败

```bash
cd neoagent && python -m pytest tests/tools/test_permission.py -v 2>&1 | head -40
```

Expected: `ModuleNotFoundError`（`neoagent.tools.permission` 不存在）。

---

### Step 6.3: 写实现 `neoagent/tools/permission.py`

- [ ] **写入以下内容到 `neoagent/tools/permission.py`：**

```python
from __future__ import annotations

from typing import Awaitable, Callable

from pydantic import BaseModel

from neoagent.tools.base import BaseTool

# 用户审批回调的类型签名：
#   tool_name   — 工具名称（便于 UI 展示）
#   description — 工具描述（帮助用户判断）
#   input_data  — 本次调用的输入参数 dict
#   返回值       — True 表示允许，False 表示拒绝
AskCallback = Callable[[str, str, dict], Awaitable[bool]]


class PermissionChecker:
    """三级权限检查器：auto / ask / deny。

    设计来源：Claude Code 三级权限体系（KB Tool System 分析）

    - auto：无条件允许，适用于只读、低风险工具（如 Read、Glob）
    - deny：无条件拒绝，适用于已禁用或危险工具
    - ask：询问用户，适用于写操作、执行类工具（如 Write、Bash）
      - 若未提供 ask_callback，默认允许（开发/测试环境友好）
      - 生产环境应注入实际的用户确认 UI 回调
    """

    def __init__(self, ask_callback: AskCallback | None = None) -> None:
        self._ask_callback = ask_callback

    async def check(self, tool: BaseTool, input: BaseModel) -> bool:
        """检查是否允许执行工具调用。

        Args:
            tool:  待执行的工具实例
            input: 已通过 input_model 验证的 Pydantic 输入实例

        Returns:
            True 表示允许执行，False 表示拒绝执行
        """
        if tool.permission == "auto":
            return True

        if tool.permission == "deny":
            return False

        # permission == "ask"
        return await self._handle_ask(tool, input)

    async def _handle_ask(self, tool: BaseTool, input: BaseModel) -> bool:
        """处理 ask 权限：调用用户注入的 callback，无 callback 时默认允许。"""
        if self._ask_callback is None:
            # 无 callback：开发/测试默认通过
            return True

        return await self._ask_callback(
            tool.name,
            tool.description,
            input.model_dump(),
        )
```

---

### Step 6.4: 运行测试，确认全部通过

```bash
cd neoagent && python -m pytest tests/tools/test_permission.py -v
```

Expected：所有测试通过，`9 passed`。

---

### Step 6.5: 运行完整工具模块测试套件，确认无回归

```bash
cd neoagent && python -m pytest tests/tools/ -v
```

Expected：全部通过，`28 passed`（Task 4 + 5 + 6 的测试合计）。

---

### Step 6.6: 提交

```bash
cd neoagent && git add neoagent/tools/permission.py tests/tools/test_permission.py
git commit -m "feat(tools): add PermissionChecker with auto/ask/deny and injectable callback"
```

---

## 验收检查单

完成 Part 2 后，执行以下命令确认整体状态：

- [ ] **全量测试（含 Part 1）：**

```bash
cd neoagent && python -m pytest tests/ -v --tb=short
```

Expected：所有测试通过，无跳过，无错误。

- [ ] **文件结构确认：**

```bash
ls neoagent/tools/
# 应输出：__init__.py  base.py  permission.py  registry.py

ls tests/tools/
# 应输出：__init__.py  test_base.py  test_permission.py  test_registry.py
```

- [ ] **类型检查（可选，pyright/mypy）：**

```bash
cd neoagent && python -m mypy neoagent/tools/ --ignore-missing-imports 2>&1 | tail -5
```

Expected：`Success: no issues found` 或仅有已知的 ABC 类型警告。

---

## 设计备注

| 决策点 | 选择 | 依据 |
|--------|------|------|
| `get_schema()` 格式 | `{name, description, input_schema}` | Anthropic API 标准格式 |
| 并发策略 | fail-closed（默认串行） | Claude Code 安全优先原则 |
| 结果截断 | 50000 字符，追加 `[truncated]` | Claude Code + Hermes 防 context 溢出 |
| `ask` 无 callback 时 | 默认 True | 开发环境友好，生产注入 callback |
| `deny` 工具 | 注册但不暴露 schema | 允许配置但禁止运行，KB 设计 |
| input 验证 | `input_model.model_validate()` | OpenHarness Pydantic 一套两用 |
