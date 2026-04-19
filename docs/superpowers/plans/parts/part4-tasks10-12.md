# neoagent v1 实现计划 — Part 4: Agent Entry + Built-in File Tools

> **For agentic workers:** Use `superpowers:subagent-driven-development` to execute this plan task-by-task.
> Each Task is independently executable. Steps use checkbox (`- [ ]`) syntax for tracking.

**范围**: Task 10 (NeoAgent 入口 + Config) + Task 11 (内置 ReadTool) + Task 12 (内置 WriteTool + EditTool)

**前置依赖**:
- `neoagent/core/types.py` — `Message`, `ToolCall`, `ToolResult`, `Turn`, `ConversationResult`
- `neoagent/core/loop.py` — `QueryLoop`
- `neoagent/core/prompt.py` — `PromptBuilder`, `PromptSection`
- `neoagent/tools/base.py` — `BaseTool`
- `neoagent/tools/registry.py` — `ToolRegistry`
- `neoagent/tools/permission.py` — `PermissionChecker`
- `neoagent/providers/anthropic.py` — `AnthropicProvider`

**目录结构**（本 Part 涉及）:
```
neoagent/
├── config.py                       # Task 10
├── agent.py                        # Task 10
└── tools/
    └── builtin/
        ├── __init__.py
        ├── read.py                 # Task 11
        ├── write.py                # Task 12
        └── edit.py                 # Task 12
tests/
├── test_agent.py                   # Task 10
└── tools/
    └── builtin/
        ├── __init__.py
        ├── test_read.py            # Task 11
        └── test_write_edit.py      # Task 12
```

---

## Task 10: Agent 入口 + Config (`neoagent/config.py`, `neoagent/agent.py`)

### Files

- `neoagent/config.py`
- `neoagent/agent.py`
- `tests/test_agent.py`

---

### Step 1 — 写失败测试

文件路径: `tests/test_agent.py`

```python
from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from neoagent.agent import NeoAgent
from neoagent.config import NeoAgentConfig
from neoagent.core.types import ConversationResult, Message, TextBlock, Turn


# ---------------------------------------------------------------------------
# NeoAgentConfig
# ---------------------------------------------------------------------------

class TestNeoAgentConfig:
    def test_defaults(self) -> None:
        cfg = NeoAgentConfig(api_key="sk-test")
        assert cfg.api_key == "sk-test"
        assert cfg.model == "claude-sonnet-4-20250514"
        assert cfg.max_turns == 30
        assert cfg.context_budget == 0
        assert cfg.max_result_size == 50_000

    def test_custom_values(self) -> None:
        cfg = NeoAgentConfig(
            api_key="sk-abc",
            model="claude-opus-4-20250514",
            max_turns=10,
            context_budget=100_000,
            max_result_size=20_000,
        )
        assert cfg.model == "claude-opus-4-20250514"
        assert cfg.max_turns == 10
        assert cfg.context_budget == 100_000
        assert cfg.max_result_size == 20_000

    def test_api_key_required(self) -> None:
        with pytest.raises(Exception):
            NeoAgentConfig()  # type: ignore[call-arg]


# ---------------------------------------------------------------------------
# NeoAgent construction
# ---------------------------------------------------------------------------

class TestNeoAgentConstruction:
    def test_constructs_from_config(self) -> None:
        with patch("neoagent.agent.AsyncAnthropic"):
            agent = NeoAgent(NeoAgentConfig(api_key="sk-test"))
        assert agent is not None

    def test_constructs_with_api_key_shorthand(self) -> None:
        """NeoAgent accepts a bare api_key string for convenience."""
        with patch("neoagent.agent.AsyncAnthropic"):
            agent = NeoAgent.from_api_key("sk-test")
        assert isinstance(agent, NeoAgent)

    def test_register_tool_delegates_to_registry(self) -> None:
        from neoagent.tools.base import BaseTool
        from neoagent.core.types import ToolResult
        from pydantic import BaseModel

        class PingInput(BaseModel):
            msg: str

        class PingTool(BaseTool):
            name = "ping"
            description = "Returns the message back"
            input_model = PingInput
            permission = "auto"
            is_concurrent_safe = True

            async def execute(self, input: PingInput) -> ToolResult:  # type: ignore[override]
                return ToolResult(call_id="", output=input.msg)

        with patch("neoagent.agent.AsyncAnthropic"):
            agent = NeoAgent(NeoAgentConfig(api_key="sk-test"))

        agent.register_tool(PingTool())
        # If tool is registered it appears in registry schemas (permission != deny)
        schemas = agent._registry.get_schemas()
        names = [s["name"] for s in schemas]
        assert "ping" in names


# ---------------------------------------------------------------------------
# NeoAgent.chat() — simplified interface
# ---------------------------------------------------------------------------

def _make_completed_result(text: str = "Done.") -> ConversationResult:
    """Helper: build a ConversationResult with a single end_turn Turn."""
    response = Message(role="assistant", content=[TextBlock(text=text)])
    turn = Turn(
        response=response,
        tool_calls=[],
        tool_results=[],
        stop_reason="end_turn",
    )
    return ConversationResult(turns=[turn], reason="completed")


class TestNeoAgentChat:
    @pytest.fixture
    def agent(self) -> NeoAgent:
        with patch("neoagent.agent.AsyncAnthropic"):
            a = NeoAgent(NeoAgentConfig(api_key="sk-test"))
        return a

    async def test_chat_returns_string(self, agent: NeoAgent) -> None:
        agent._loop.run = AsyncMock(return_value=_make_completed_result("Hello!"))  # type: ignore[method-assign]
        result = await agent.chat("hi")
        assert isinstance(result, str)
        assert result == "Hello!"

    async def test_chat_constructs_user_message(self, agent: NeoAgent) -> None:
        captured: list[list[Message]] = []

        async def capture_run(messages: list[Message]) -> ConversationResult:
            captured.append(messages)
            return _make_completed_result()

        agent._loop.run = capture_run  # type: ignore[method-assign]
        await agent.chat("what is 2+2?")

        assert len(captured) == 1
        msgs = captured[0]
        assert len(msgs) == 1
        assert msgs[0].role == "user"
        assert msgs[0].content == "what is 2+2?"

    async def test_chat_extracts_last_text_block(self, agent: NeoAgent) -> None:
        """chat() returns text from the last turn's response text blocks."""
        response = Message(
            role="assistant",
            content=[TextBlock(text="Part 1"), TextBlock(text="Part 2")],
        )
        turn = Turn(
            response=response,
            tool_calls=[],
            tool_results=[],
            stop_reason="end_turn",
        )
        result = ConversationResult(turns=[turn], reason="completed")
        agent._loop.run = AsyncMock(return_value=result)  # type: ignore[method-assign]

        text = await agent.chat("go")
        assert text == "Part 1\nPart 2"

    async def test_chat_max_turns_reason(self, agent: NeoAgent) -> None:
        """chat() still returns text when reason is max_turns."""
        response = Message(role="assistant", content=[TextBlock(text="Partial answer")])
        turn = Turn(
            response=response,
            tool_calls=[],
            tool_results=[],
            stop_reason="end_turn",
        )
        result = ConversationResult(turns=[turn], reason="max_turns")
        agent._loop.run = AsyncMock(return_value=result)  # type: ignore[method-assign]

        text = await agent.chat("complex task")
        assert text == "Partial answer"


# ---------------------------------------------------------------------------
# NeoAgent.run() — low-level interface
# ---------------------------------------------------------------------------

class TestNeoAgentRun:
    @pytest.fixture
    def agent(self) -> NeoAgent:
        with patch("neoagent.agent.AsyncAnthropic"):
            a = NeoAgent(NeoAgentConfig(api_key="sk-test"))
        return a

    async def test_run_delegates_to_loop(self, agent: NeoAgent) -> None:
        expected = _make_completed_result("result")
        agent._loop.run = AsyncMock(return_value=expected)  # type: ignore[method-assign]

        messages = [Message(role="user", content="run this")]
        result = await agent.run(messages)

        assert result is expected
        agent._loop.run.assert_awaited_once_with(messages)

    async def test_run_returns_conversation_result(self, agent: NeoAgent) -> None:
        agent._loop.run = AsyncMock(return_value=_make_completed_result())  # type: ignore[method-assign]
        result = await agent.run([Message(role="user", content="x")])
        assert isinstance(result, ConversationResult)
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/test_agent.py -v`
Expected: FAIL with `ImportError: cannot import name 'NeoAgentConfig' from 'neoagent.config'`

- [ ] **Step 3: Write implementation**

`neoagent/neoagent/config.py`
```python
from __future__ import annotations

from dataclasses import dataclass, field


@dataclass
class NeoAgentConfig:
    """Top-level configuration for a NeoAgent instance.

    Attributes:
        api_key:         Anthropic API key (required).
        model:           Model identifier passed to the Anthropic API.
        max_turns:       Hard upper bound on agent loop iterations.
        context_budget:  Token budget for context; 0 means auto-detect from
                         the provider's context window.
        max_result_size: Maximum bytes returned by a single tool execution
                         before the result is truncated.
    """

    api_key: str
    model: str = "claude-sonnet-4-20250514"
    max_turns: int = 30
    context_budget: int = 0
    max_result_size: int = 50_000
```

`neoagent/neoagent/agent.py`
```python
from __future__ import annotations

from anthropic import AsyncAnthropic  # noqa: F401 — imported so tests can patch it

from neoagent.config import NeoAgentConfig
from neoagent.core.loop import QueryLoop
from neoagent.core.prompt import PromptBuilder, PromptSection
from neoagent.core.types import ConversationResult, Message, TextBlock
from neoagent.providers.anthropic import AnthropicProvider
from neoagent.tools.permission import PermissionChecker
from neoagent.tools.registry import ToolRegistry
from neoagent.tools.base import BaseTool


class NeoAgent:
    """Top-level agent entry point.

    Assembles all v1 subsystems (provider, registry, prompt builder, query loop)
    and exposes two public interfaces:

    - ``chat(message)`` — simplified: accepts a string, returns a string.
    - ``run(messages)`` — low-level: full message list in, ConversationResult out.
    """

    def __init__(self, config: NeoAgentConfig) -> None:
        self._config = config

        # --- provider ---
        self._provider = AnthropicProvider(
            api_key=config.api_key,
            model=config.model,
        )

        # --- tool registry + permission checker ---
        self._checker = PermissionChecker()
        self._registry = ToolRegistry(permission_checker=self._checker)

        # --- prompt builder with sensible defaults ---
        self._prompt_builder = PromptBuilder()
        self._prompt_builder.add_section(
            PromptSection(
                name="identity",
                content="You are neoagent, a capable AI assistant.",
                priority=0,
                is_static=True,
            )
        )

        # --- resolve context budget ---
        context_budget = config.context_budget
        if context_budget == 0:
            context_budget = self._provider.get_context_window()

        # --- query loop ---
        self._loop = QueryLoop(
            provider=self._provider,
            tool_registry=self._registry,
            prompt_builder=self._prompt_builder,
            max_turns=config.max_turns,
            context_budget=context_budget,
        )

    # ------------------------------------------------------------------
    # Class-method constructors
    # ------------------------------------------------------------------

    @classmethod
    def from_api_key(cls, api_key: str, **kwargs: object) -> NeoAgent:
        """Convenience constructor: create an agent from a bare API key.

        Any additional keyword arguments are forwarded to NeoAgentConfig.
        """
        return cls(NeoAgentConfig(api_key=api_key, **kwargs))  # type: ignore[arg-type]

    # ------------------------------------------------------------------
    # Tool registration
    # ------------------------------------------------------------------

    def register_tool(self, tool: BaseTool) -> None:
        """Register a tool with the agent's ToolRegistry."""
        self._registry.register(tool)

    # ------------------------------------------------------------------
    # Public interfaces
    # ------------------------------------------------------------------

    async def chat(self, message: str) -> str:
        """Simplified chat interface.

        Wraps ``message`` in a user Message, runs the agent loop, and
        returns the final assistant response as a plain string.
        """
        messages = [Message(role="user", content=message)]
        result = await self._loop.run(messages)
        return self._extract_text(result)

    async def run(self, messages: list[Message]) -> ConversationResult:
        """Low-level interface: pass a full message list, get a ConversationResult."""
        return await self._loop.run(messages)

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    @staticmethod
    def _extract_text(result: ConversationResult) -> str:
        """Extract the final assistant text from a ConversationResult.

        Joins all TextBlock texts from the last turn's response with newlines.
        Returns an empty string if no turns or no text blocks.
        """
        if not result.turns:
            return ""
        last_turn = result.turns[-1]
        content = last_turn.response.content
        if isinstance(content, str):
            return content
        parts = [block.text for block in content if isinstance(block, TextBlock)]
        return "\n".join(parts)
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/test_agent.py -v`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add neoagent/neoagent/config.py \
        neoagent/neoagent/agent.py \
        neoagent/tests/test_agent.py
git commit -m "feat: add NeoAgentConfig and NeoAgent entry point with chat/run interfaces"
```

---

## Task 11: 内置工具 — Read (`neoagent/tools/builtin/read.py`)

### Files

- `neoagent/tools/builtin/__init__.py`  *(already exists from Task 1 scaffold)*
- `neoagent/tools/builtin/read.py`
- `tests/tools/builtin/__init__.py`
- `tests/tools/builtin/test_read.py`

---

### Step 1 — 写失败测试

文件路径: `tests/tools/builtin/__init__.py`

```python
from __future__ import annotations
```

文件路径: `tests/tools/builtin/test_read.py`

```python
from __future__ import annotations

import textwrap
from pathlib import Path

import pytest

from neoagent.core.types import ToolResult
from neoagent.tools.builtin.read import ReadInput, ReadTool


# ---------------------------------------------------------------------------
# ReadInput validation
# ---------------------------------------------------------------------------

class TestReadInput:
    def test_defaults(self) -> None:
        inp = ReadInput(file_path="/tmp/file.txt")
        assert inp.file_path == "/tmp/file.txt"
        assert inp.offset == 0
        assert inp.limit == 2000

    def test_custom_offset_and_limit(self) -> None:
        inp = ReadInput(file_path="/f", offset=10, limit=50)
        assert inp.offset == 10
        assert inp.limit == 50

    def test_negative_offset_rejected(self) -> None:
        with pytest.raises(Exception):
            ReadInput(file_path="/f", offset=-1)

    def test_zero_limit_rejected(self) -> None:
        with pytest.raises(Exception):
            ReadInput(file_path="/f", limit=0)


# ---------------------------------------------------------------------------
# ReadTool metadata
# ---------------------------------------------------------------------------

class TestReadToolMetadata:
    def test_name(self) -> None:
        assert ReadTool.name == "read"

    def test_permission_is_auto(self) -> None:
        assert ReadTool.permission == "auto"

    def test_is_concurrent_safe(self) -> None:
        assert ReadTool.is_concurrent_safe is True

    def test_description_nonempty(self) -> None:
        assert len(ReadTool.description) > 0

    def test_input_model_is_read_input(self) -> None:
        assert ReadTool.input_model is ReadInput


# ---------------------------------------------------------------------------
# ReadTool.execute() — happy path
# ---------------------------------------------------------------------------

class TestReadToolExecute:
    @pytest.fixture
    def tmp_file(self, tmp_path: Path) -> Path:
        f = tmp_path / "sample.txt"
        # 5 lines of content
        lines = [f"line {i}: hello world" for i in range(1, 6)]
        f.write_text("\n".join(lines))
        return f

    async def test_reads_file_with_line_numbers(self, tmp_file: Path) -> None:
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(tmp_file)))
        assert result.is_error is False
        # Line-number prefix format: "<N>\t<content>"
        assert "1\t" in result.output
        assert "line 1: hello world" in result.output

    async def test_all_lines_present(self, tmp_file: Path) -> None:
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(tmp_file)))
        for i in range(1, 6):
            assert f"line {i}: hello world" in result.output

    async def test_offset_skips_lines(self, tmp_file: Path) -> None:
        """offset=2 should skip the first 2 lines (0-indexed)."""
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(tmp_file), offset=2))
        assert result.is_error is False
        # Line 1 and 2 should not appear
        assert "line 1: hello world" not in result.output
        assert "line 2: hello world" not in result.output
        # Lines 3-5 should be present
        assert "line 3: hello world" in result.output

    async def test_limit_restricts_lines(self, tmp_file: Path) -> None:
        """limit=2 should return at most 2 lines."""
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(tmp_file), limit=2))
        assert result.is_error is False
        lines = [l for l in result.output.splitlines() if l.strip()]
        assert len(lines) <= 2

    async def test_offset_and_limit_combined(self, tmp_file: Path) -> None:
        """offset=1, limit=2 — skip first line, take next 2."""
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(tmp_file), offset=1, limit=2))
        assert result.is_error is False
        # Should contain line 2 and line 3
        assert "line 2: hello world" in result.output
        assert "line 3: hello world" in result.output
        # Should NOT contain line 1 or line 4
        assert "line 1: hello world" not in result.output
        assert "line 4: hello world" not in result.output

    async def test_line_number_prefix_format(self, tmp_file: Path) -> None:
        """Each line must start with '<line_number>\\t'."""
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(tmp_file)))
        for raw_line in result.output.splitlines():
            if raw_line.strip():
                num_str, _, _ = raw_line.partition("\t")
                assert num_str.isdigit(), f"Expected numeric prefix, got: {raw_line!r}"

    async def test_line_numbers_start_at_offset_plus_one(self, tmp_file: Path) -> None:
        """When offset=2, line numbers in output should start at 3."""
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(tmp_file), offset=2))
        first_line = result.output.splitlines()[0]
        num_str = first_line.split("\t")[0]
        assert int(num_str) == 3

    async def test_reads_empty_file(self, tmp_path: Path) -> None:
        empty = tmp_path / "empty.txt"
        empty.write_text("")
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(empty)))
        assert result.is_error is False
        assert result.output == ""


# ---------------------------------------------------------------------------
# ReadTool.execute() — error handling
# ---------------------------------------------------------------------------

class TestReadToolErrors:
    async def test_file_not_found_returns_error(self) -> None:
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path="/nonexistent/path/file.txt"))
        assert result.is_error is True
        assert "not found" in result.output.lower() or "no such file" in result.output.lower()

    async def test_error_result_contains_path(self) -> None:
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path="/does/not/exist.txt"))
        assert "/does/not/exist.txt" in result.output

    async def test_is_a_directory_returns_error(self, tmp_path: Path) -> None:
        tool = ReadTool()
        result = await tool.execute(ReadInput(file_path=str(tmp_path)))
        assert result.is_error is True
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/tools/builtin/test_read.py -v`
Expected: FAIL with `ImportError: cannot import name 'ReadInput' from 'neoagent.tools.builtin.read'`

- [ ] **Step 3: Write implementation**

`neoagent/neoagent/tools/builtin/read.py`
```python
from __future__ import annotations

from pathlib import Path

from pydantic import BaseModel, Field

from neoagent.core.types import ToolResult
from neoagent.tools.base import BaseTool


class ReadInput(BaseModel):
    """Input schema for the Read tool."""

    file_path: str = Field(..., description="Absolute path to the file to read.")
    offset: int = Field(
        default=0,
        ge=0,
        description="Number of lines to skip from the beginning of the file (0-indexed).",
    )
    limit: int = Field(
        default=2000,
        gt=0,
        description="Maximum number of lines to return.",
    )


class ReadTool(BaseTool):
    """Read a file and return its contents with 1-based line number prefixes.

    Supports offset/limit slicing for large files. Permission is 'auto' —
    no user confirmation required for read-only operations.
    """

    name = "read"
    description = "读取文件内容，返回带行号的文本（支持 offset/limit 切片）"
    input_model = ReadInput
    permission = "auto"
    is_concurrent_safe = True

    async def execute(self, input: ReadInput) -> ToolResult:  # type: ignore[override]
        path = Path(input.file_path)

        # --- existence and type checks ---
        if not path.exists():
            return ToolResult(
                call_id="",
                output=f"File not found: {input.file_path}",
                is_error=True,
            )
        if path.is_dir():
            return ToolResult(
                call_id="",
                output=f"Path is a directory, not a file: {input.file_path}",
                is_error=True,
            )

        # --- read and slice ---
        try:
            all_lines = path.read_text(encoding="utf-8", errors="replace").splitlines()
        except OSError as exc:
            return ToolResult(
                call_id="",
                output=f"Cannot read {input.file_path}: {exc}",
                is_error=True,
            )

        sliced = all_lines[input.offset : input.offset + input.limit]

        if not sliced:
            return ToolResult(call_id="", output="")

        # --- add 1-based line numbers (relative to original file) ---
        numbered_lines = [
            f"{input.offset + idx + 1}\t{line}"
            for idx, line in enumerate(sliced)
        ]
        return ToolResult(call_id="", output="\n".join(numbered_lines))
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/tools/builtin/test_read.py -v`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add neoagent/neoagent/tools/builtin/read.py \
        neoagent/tests/tools/builtin/__init__.py \
        neoagent/tests/tools/builtin/test_read.py
git commit -m "feat: add ReadTool builtin — offset/limit slicing with line-number prefixes"
```

---

## Task 12: 内置工具 — Write + Edit (`neoagent/tools/builtin/write.py`, `edit.py`)

### Files

- `neoagent/tools/builtin/write.py`
- `neoagent/tools/builtin/edit.py`
- `tests/tools/builtin/test_write_edit.py`

---

### Step 1 — 写失败测试

文件路径: `tests/tools/builtin/test_write_edit.py`

```python
from __future__ import annotations

from pathlib import Path

import pytest

from neoagent.core.types import ToolResult
from neoagent.tools.builtin.edit import EditInput, EditTool
from neoagent.tools.builtin.write import WriteInput, WriteTool


# ===========================================================================
# WriteTool
# ===========================================================================

class TestWriteInput:
    def test_construction(self) -> None:
        inp = WriteInput(file_path="/tmp/out.txt", content="hello")
        assert inp.file_path == "/tmp/out.txt"
        assert inp.content == "hello"

    def test_file_path_required(self) -> None:
        with pytest.raises(Exception):
            WriteInput(content="x")  # type: ignore[call-arg]

    def test_content_required(self) -> None:
        with pytest.raises(Exception):
            WriteInput(file_path="/f")  # type: ignore[call-arg]


class TestWriteToolMetadata:
    def test_name(self) -> None:
        assert WriteTool.name == "write"

    def test_permission_is_ask(self) -> None:
        assert WriteTool.permission == "ask"

    def test_is_not_concurrent_safe(self) -> None:
        assert WriteTool.is_concurrent_safe is False

    def test_input_model_is_write_input(self) -> None:
        assert WriteTool.input_model is WriteInput


class TestWriteToolExecute:
    async def test_write_new_file(self, tmp_path: Path) -> None:
        target = tmp_path / "new_file.txt"
        tool = WriteTool()
        result = await tool.execute(WriteInput(file_path=str(target), content="hello world"))
        assert result.is_error is False
        assert target.read_text() == "hello world"

    async def test_write_overwrites_existing_file(self, tmp_path: Path) -> None:
        target = tmp_path / "existing.txt"
        target.write_text("old content")
        tool = WriteTool()
        result = await tool.execute(WriteInput(file_path=str(target), content="new content"))
        assert result.is_error is False
        assert target.read_text() == "new content"

    async def test_write_creates_parent_directories(self, tmp_path: Path) -> None:
        target = tmp_path / "a" / "b" / "c" / "nested.txt"
        tool = WriteTool()
        result = await tool.execute(WriteInput(file_path=str(target), content="deep"))
        assert result.is_error is False
        assert target.read_text() == "deep"

    async def test_write_empty_content(self, tmp_path: Path) -> None:
        target = tmp_path / "empty.txt"
        tool = WriteTool()
        result = await tool.execute(WriteInput(file_path=str(target), content=""))
        assert result.is_error is False
        assert target.read_text() == ""

    async def test_write_result_contains_path(self, tmp_path: Path) -> None:
        target = tmp_path / "out.txt"
        tool = WriteTool()
        result = await tool.execute(WriteInput(file_path=str(target), content="x"))
        assert str(target) in result.output

    async def test_write_multiline_content(self, tmp_path: Path) -> None:
        content = "line1\nline2\nline3"
        target = tmp_path / "multi.txt"
        tool = WriteTool()
        await tool.execute(WriteInput(file_path=str(target), content=content))
        assert target.read_text() == content


# ===========================================================================
# EditTool
# ===========================================================================

class TestEditInput:
    def test_construction(self) -> None:
        inp = EditInput(
            file_path="/tmp/f.txt",
            old_string="foo",
            new_string="bar",
        )
        assert inp.file_path == "/tmp/f.txt"
        assert inp.old_string == "foo"
        assert inp.new_string == "bar"

    def test_all_fields_required(self) -> None:
        with pytest.raises(Exception):
            EditInput(file_path="/f", old_string="x")  # type: ignore[call-arg]


class TestEditToolMetadata:
    def test_name(self) -> None:
        assert EditTool.name == "edit"

    def test_permission_is_ask(self) -> None:
        assert EditTool.permission == "ask"

    def test_is_not_concurrent_safe(self) -> None:
        assert EditTool.is_concurrent_safe is False

    def test_input_model_is_edit_input(self) -> None:
        assert EditTool.input_model is EditInput


class TestEditToolExecute:
    async def test_basic_replacement(self, tmp_path: Path) -> None:
        target = tmp_path / "code.py"
        target.write_text("def foo():\n    return 1\n")
        tool = EditTool()
        result = await tool.execute(
            EditInput(
                file_path=str(target),
                old_string="return 1",
                new_string="return 42",
            )
        )
        assert result.is_error is False
        assert "return 42" in target.read_text()
        assert "return 1" not in target.read_text()

    async def test_replacement_preserves_surrounding_content(self, tmp_path: Path) -> None:
        original = "line one\nTARGET\nline three"
        target = tmp_path / "text.txt"
        target.write_text(original)
        tool = EditTool()
        await tool.execute(
            EditInput(file_path=str(target), old_string="TARGET", new_string="REPLACED")
        )
        content = target.read_text()
        assert "line one" in content
        assert "REPLACED" in content
        assert "line three" in content

    async def test_multiline_old_string(self, tmp_path: Path) -> None:
        original = "start\nfoo\nbar\nend"
        target = tmp_path / "multi.txt"
        target.write_text(original)
        tool = EditTool()
        result = await tool.execute(
            EditInput(
                file_path=str(target),
                old_string="foo\nbar",
                new_string="baz\nqux",
            )
        )
        assert result.is_error is False
        content = target.read_text()
        assert "baz\nqux" in content
        assert "foo\nbar" not in content

    async def test_result_contains_path(self, tmp_path: Path) -> None:
        target = tmp_path / "f.txt"
        target.write_text("hello world")
        tool = EditTool()
        result = await tool.execute(
            EditInput(file_path=str(target), old_string="hello", new_string="hi")
        )
        assert str(target) in result.output

    # --- error cases ---

    async def test_file_not_found(self) -> None:
        tool = EditTool()
        result = await tool.execute(
            EditInput(
                file_path="/nonexistent/path/file.py",
                old_string="x",
                new_string="y",
            )
        )
        assert result.is_error is True
        assert "not found" in result.output.lower() or "no such file" in result.output.lower()

    async def test_old_string_not_found(self, tmp_path: Path) -> None:
        target = tmp_path / "code.txt"
        target.write_text("hello world")
        tool = EditTool()
        result = await tool.execute(
            EditInput(
                file_path=str(target),
                old_string="this does not exist",
                new_string="replacement",
            )
        )
        assert result.is_error is True
        assert "not found" in result.output.lower() or "does not exist" in result.output.lower()

    async def test_old_string_not_unique_raises_error(self, tmp_path: Path) -> None:
        """old_string appearing more than once must return is_error=True."""
        target = tmp_path / "dup.txt"
        target.write_text("foo\nfoo\nother content")
        tool = EditTool()
        result = await tool.execute(
            EditInput(
                file_path=str(target),
                old_string="foo",
                new_string="bar",
            )
        )
        assert result.is_error is True
        assert "unique" in result.output.lower() or "multiple" in result.output.lower() or "ambiguous" in result.output.lower()

    async def test_file_unchanged_when_old_string_not_found(self, tmp_path: Path) -> None:
        original = "unchanged content"
        target = tmp_path / "f.txt"
        target.write_text(original)
        tool = EditTool()
        await tool.execute(
            EditInput(file_path=str(target), old_string="MISSING", new_string="x")
        )
        assert target.read_text() == original

    async def test_file_unchanged_when_not_unique(self, tmp_path: Path) -> None:
        original = "dup\ndup\n"
        target = tmp_path / "f.txt"
        target.write_text(original)
        tool = EditTool()
        await tool.execute(
            EditInput(file_path=str(target), old_string="dup", new_string="x")
        )
        assert target.read_text() == original
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/tools/builtin/test_write_edit.py -v`
Expected: FAIL with `ImportError: cannot import name 'WriteInput' from 'neoagent.tools.builtin.write'`

- [ ] **Step 3: Write implementation**

`neoagent/neoagent/tools/builtin/write.py`
```python
from __future__ import annotations

from pathlib import Path

from pydantic import BaseModel, Field

from neoagent.core.types import ToolResult
from neoagent.tools.base import BaseTool


class WriteInput(BaseModel):
    """Input schema for the Write tool."""

    file_path: str = Field(..., description="Absolute path of the file to write.")
    content: str = Field(..., description="Full content to write into the file.")


class WriteTool(BaseTool):
    """Write (or overwrite) a file with the given content.

    Parent directories are created automatically. Permission is 'ask' —
    the user must confirm each write operation in interactive sessions.
    """

    name = "write"
    description = "将内容写入文件（自动创建父目录，已有文件直接覆盖）"
    input_model = WriteInput
    permission = "ask"
    is_concurrent_safe = False

    async def execute(self, input: WriteInput) -> ToolResult:  # type: ignore[override]
        path = Path(input.file_path)

        try:
            path.parent.mkdir(parents=True, exist_ok=True)
            path.write_text(input.content, encoding="utf-8")
        except OSError as exc:
            return ToolResult(
                call_id="",
                output=f"Failed to write {input.file_path}: {exc}",
                is_error=True,
            )

        return ToolResult(
            call_id="",
            output=f"Written {len(input.content)} bytes to {input.file_path}",
        )
```

`neoagent/neoagent/tools/builtin/edit.py`
```python
from __future__ import annotations

from pathlib import Path

from pydantic import BaseModel, Field

from neoagent.core.types import ToolResult
from neoagent.tools.base import BaseTool


class EditInput(BaseModel):
    """Input schema for the Edit tool."""

    file_path: str = Field(..., description="Absolute path of the file to edit.")
    old_string: str = Field(
        ...,
        description=(
            "The exact string to find and replace. "
            "Must appear exactly once in the file."
        ),
    )
    new_string: str = Field(..., description="The string to substitute in place of old_string.")


class EditTool(BaseTool):
    """Perform an exact string replacement within a file.

    Rules enforced:
    - The file must exist.
    - ``old_string`` must be present in the file (otherwise is_error=True).
    - ``old_string`` must appear exactly once (ambiguous replacements are rejected).

    Permission is 'ask' — the user must confirm each edit in interactive sessions.
    The file is not modified if any rule is violated.
    """

    name = "edit"
    description = "在文件中执行精确字符串替换（old_string 必须唯一存在）"
    input_model = EditInput
    permission = "ask"
    is_concurrent_safe = False

    async def execute(self, input: EditInput) -> ToolResult:  # type: ignore[override]
        path = Path(input.file_path)

        # --- existence check ---
        if not path.exists():
            return ToolResult(
                call_id="",
                output=f"File not found: {input.file_path}",
                is_error=True,
            )

        # --- read ---
        try:
            content = path.read_text(encoding="utf-8", errors="replace")
        except OSError as exc:
            return ToolResult(
                call_id="",
                output=f"Cannot read {input.file_path}: {exc}",
                is_error=True,
            )

        # --- uniqueness check ---
        count = content.count(input.old_string)
        if count == 0:
            return ToolResult(
                call_id="",
                output=(
                    f"old_string does not exist in {input.file_path}. "
                    "No changes made."
                ),
                is_error=True,
            )
        if count > 1:
            return ToolResult(
                call_id="",
                output=(
                    f"old_string is not unique in {input.file_path} "
                    f"(found {count} occurrences). "
                    "Provide more context to make it unambiguous. No changes made."
                ),
                is_error=True,
            )

        # --- replace and write ---
        new_content = content.replace(input.old_string, input.new_string, 1)
        try:
            path.write_text(new_content, encoding="utf-8")
        except OSError as exc:
            return ToolResult(
                call_id="",
                output=f"Failed to write {input.file_path}: {exc}",
                is_error=True,
            )

        return ToolResult(
            call_id="",
            output=f"Edited {input.file_path}: replaced 1 occurrence.",
        )
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/tools/builtin/test_write_edit.py -v`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add neoagent/neoagent/tools/builtin/write.py \
        neoagent/neoagent/tools/builtin/edit.py \
        neoagent/tests/tools/builtin/test_write_edit.py
git commit -m "feat: add WriteTool and EditTool builtins with uniqueness enforcement"
```

---

## 完成检查

全部三个 Task 完成后运行完整测试套件确认无回归：

```bash
cd neoagent && python -m pytest tests/ -v --tb=short
```

预期：Task 10、11、12 的所有测试全部绿色通过，无新失败。
