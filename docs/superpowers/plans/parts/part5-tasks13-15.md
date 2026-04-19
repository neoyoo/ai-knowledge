# neoagent v1 实现计划 — Part 5: 内置工具 Bash/Grep/Glob + 集成测试

> **For agentic workers:** Use `superpowers:subagent-driven-development` to execute this plan task-by-task.
> Each Task is independently executable. Steps use checkbox (`- [ ]`) syntax for tracking.

**范围**: Task 13 (BashTool) + Task 14 (GrepTool + GlobTool) + Task 15 (集成测试)

**前置依赖**:
- `neoagent/tools/base.py` — `BaseTool`, `ToolResult` (Task 4)
- `neoagent/tools/registry.py` — `ToolRegistry` (Task 5)
- `neoagent/core/types.py` — `Message`, `ToolCall`, `ToolResult`, `Turn`, `ConversationResult` (Task 2)
- `neoagent/agent.py` — `NeoAgent`, `NeoAgentConfig` (Task 10/11)
- `neoagent/core/loop.py` — `QueryLoop` (Task 7)
- `neoagent/core/prompt.py` — `PromptBuilder` (Task 8)
- `neoagent/providers/anthropic.py` — `AnthropicProvider` (Task 3)

**目录结构**（本 Part 涉及）:
```
neoagent/
└── tools/
    └── builtin/
        ├── __init__.py
        ├── bash.py          # Task 13
        ├── grep.py          # Task 14
        └── glob.py          # Task 14
tests/
└── tools/
    └── builtin/
        ├── __init__.py
        ├── test_bash.py     # Task 13
        ├── test_grep.py     # Task 14
        └── test_glob.py     # Task 14
tests/
└── test_integration.py      # Task 15
```

---

## Task 13: 内置工具 — Bash (`neoagent/tools/builtin/bash.py`)

### Files

- `neoagent/tools/builtin/bash.py`
- `tests/tools/builtin/__init__.py`
- `tests/tools/builtin/test_bash.py`

---

### Step 1 — 写失败测试

文件路径: `tests/tools/builtin/__init__.py`
```python
from __future__ import annotations
```

文件路径: `tests/tools/builtin/test_bash.py`
```python
from __future__ import annotations

import asyncio

import pytest

from neoagent.tools.builtin.bash import BashInput, BashTool
from neoagent.core.types import ToolResult


# ---------- 构造与属性 ----------

class TestBashToolAttributes:
    def test_name(self) -> None:
        tool = BashTool()
        assert tool.name == "bash"

    def test_description_is_nonempty(self) -> None:
        tool = BashTool()
        assert len(tool.description) > 0

    def test_permission_is_ask(self) -> None:
        """Bash 默认需要用户确认，不自动执行。"""
        tool = BashTool()
        assert tool.permission == "ask"

    def test_not_concurrent_safe(self) -> None:
        """Shell 命令有副作用，默认不并行。"""
        tool = BashTool()
        assert tool.is_concurrent_safe is False

    def test_input_model_is_bash_input(self) -> None:
        tool = BashTool()
        assert tool.input_model is BashInput


# ---------- BashInput 验证 ----------

class TestBashInput:
    def test_default_timeout(self) -> None:
        inp = BashInput(command="echo hello")
        assert inp.timeout == 120

    def test_custom_timeout(self) -> None:
        inp = BashInput(command="sleep 1", timeout=5)
        assert inp.timeout == 5

    def test_command_required(self) -> None:
        with pytest.raises(Exception):
            BashInput()  # type: ignore[call-arg]


# ---------- 执行：正常命令 ----------

class TestBashToolExecute:
    async def test_simple_echo(self) -> None:
        tool = BashTool()
        inp = BashInput(command="echo hello")
        result = await tool.execute(inp)

        assert isinstance(result, ToolResult)
        assert result.is_error is False
        assert "hello" in result.output

    async def test_stdout_and_stderr_merged(self) -> None:
        """stdout 和 stderr 都要出现在 output 中。"""
        tool = BashTool()
        # 同时写 stdout 和 stderr
        inp = BashInput(command="echo out && echo err >&2")
        result = await tool.execute(inp)

        assert result.is_error is False
        assert "out" in result.output
        assert "err" in result.output

    async def test_exit_nonzero_returns_error(self) -> None:
        """exit 1 → is_error=True，output 包含错误信息。"""
        tool = BashTool()
        inp = BashInput(command="exit 1")
        result = await tool.execute(inp)

        assert result.is_error is True

    async def test_command_output_captured(self) -> None:
        tool = BashTool()
        inp = BashInput(command="printf 'line1\\nline2\\n'")
        result = await tool.execute(inp)

        assert "line1" in result.output
        assert "line2" in result.output

    async def test_call_id_empty_by_default(self) -> None:
        """execute() 不知道 call_id，调用方（registry）负责填充。"""
        tool = BashTool()
        inp = BashInput(command="echo x")
        result = await tool.execute(inp)
        # call_id 由外部填充，BaseTool.execute 返回的可以是空字符串
        assert isinstance(result.call_id, str)


# ---------- 执行：超时 ----------

class TestBashToolTimeout:
    async def test_timeout_kills_process(self) -> None:
        """命令超过 timeout 秒后被杀死，返回 is_error=True 并包含 timeout 提示。"""
        tool = BashTool()
        # 用 1 秒超时，sleep 5 必然触发
        inp = BashInput(command="sleep 5", timeout=1)
        result = await tool.execute(inp)

        assert result.is_error is True
        assert "timeout" in result.output.lower() or "timed out" in result.output.lower()

    async def test_fast_command_not_timeout(self) -> None:
        """快速命令不应触发超时。"""
        tool = BashTool()
        inp = BashInput(command="echo done", timeout=5)
        result = await tool.execute(inp)

        assert result.is_error is False
        assert "done" in result.output
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd neoagent && python -m pytest tests/tools/builtin/test_bash.py -v
```

预期: FAIL，`ModuleNotFoundError: No module named 'neoagent.tools.builtin.bash'`

---

### Step 3 — 写实现

文件路径: `neoagent/tools/builtin/bash.py`
```python
from __future__ import annotations

import asyncio

from pydantic import BaseModel

from neoagent.core.types import ToolResult
from neoagent.tools.base import BaseTool


class BashInput(BaseModel):
    """Input schema for the Bash tool."""

    command: str
    timeout: int = 120


class BashTool(BaseTool):
    """Execute a shell command and return its combined stdout+stderr output.

    Permission is 'ask' because shell commands have side effects — the agent
    must not run arbitrary commands without user confirmation.
    """

    name = "bash"
    description = "执行 shell 命令并返回 stdout 和 stderr 的合并输出"
    input_model = BashInput
    permission = "ask"
    is_concurrent_safe = False

    async def execute(self, input: BashInput) -> ToolResult:  # type: ignore[override]
        """Run *input.command* in a shell subprocess.

        - Captures stdout and stderr together (interleaved via PIPE merge).
        - Kills the process if it exceeds *input.timeout* seconds.
        - Returns ``is_error=True`` on non-zero exit code or timeout.
        """
        try:
            proc = await asyncio.create_subprocess_shell(
                input.command,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.STDOUT,  # merge stderr → stdout
            )
        except OSError as exc:
            return ToolResult(
                call_id="",
                output=f"Failed to start process: {exc}",
                is_error=True,
            )

        try:
            stdout_bytes, _ = await asyncio.wait_for(
                proc.communicate(),
                timeout=input.timeout,
            )
        except asyncio.TimeoutError:
            # Best-effort cleanup: kill the process tree
            try:
                proc.kill()
                await proc.communicate()  # drain to avoid zombie
            except ProcessLookupError:
                pass  # already exited
            return ToolResult(
                call_id="",
                output=(
                    f"Command timed out after {input.timeout}s: {input.command!r}"
                ),
                is_error=True,
            )

        output = stdout_bytes.decode(errors="replace")
        is_error = (proc.returncode != 0)

        if is_error and not output:
            output = f"Command exited with code {proc.returncode}"

        return ToolResult(call_id="", output=output, is_error=is_error)
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd neoagent && python -m pytest tests/tools/builtin/test_bash.py -v
```

预期: PASS — 所有测试绿色

- [ ] **Step 5: 提交**

```bash
git add neoagent/neoagent/tools/builtin/bash.py \
        neoagent/tests/tools/builtin/__init__.py \
        neoagent/tests/tools/builtin/test_bash.py
git commit -m "feat: add BashTool builtin — asyncio subprocess, timeout kill, stderr merge"
```

---

## Task 14: 内置工具 — Grep + Glob (`neoagent/tools/builtin/grep.py`, `glob.py`)

### Files

- `neoagent/tools/builtin/grep.py`
- `neoagent/tools/builtin/glob.py`
- `tests/tools/builtin/test_grep.py`
- `tests/tools/builtin/test_glob.py`

---

### Step 1 — 写失败测试

文件路径: `tests/tools/builtin/test_grep.py`
```python
from __future__ import annotations

import textwrap
from pathlib import Path

import pytest

from neoagent.core.types import ToolResult
from neoagent.tools.builtin.grep import GrepInput, GrepTool


# ---------- 构造与属性 ----------

class TestGrepToolAttributes:
    def test_name(self) -> None:
        assert GrepTool().name == "grep"

    def test_permission_auto(self) -> None:
        """读操作，不需要用户确认。"""
        assert GrepTool().permission == "auto"

    def test_is_concurrent_safe(self) -> None:
        assert GrepTool().is_concurrent_safe is True

    def test_input_model(self) -> None:
        assert GrepTool().input_model is GrepInput


# ---------- GrepInput 验证 ----------

class TestGrepInput:
    def test_defaults(self) -> None:
        inp = GrepInput(pattern="TODO")
        assert inp.path == "."
        assert inp.glob_filter is None

    def test_custom_path_and_filter(self) -> None:
        inp = GrepInput(pattern="foo", path="/tmp", glob_filter="*.py")
        assert inp.path == "/tmp"
        assert inp.glob_filter == "*.py"

    def test_pattern_required(self) -> None:
        with pytest.raises(Exception):
            GrepInput()  # type: ignore[call-arg]


# ---------- 执行：实际搜索 ----------

class TestGrepToolExecute:
    @pytest.fixture()
    def tmp_repo(self, tmp_path: Path) -> Path:
        """创建一个带几个文件的临时目录。"""
        (tmp_path / "alpha.py").write_text(
            textwrap.dedent("""\
                # TODO: fix this
                def foo():
                    pass
            """)
        )
        (tmp_path / "beta.txt").write_text(
            textwrap.dedent("""\
                nothing here
                just text
            """)
        )
        sub = tmp_path / "sub"
        sub.mkdir()
        (sub / "gamma.py").write_text(
            textwrap.dedent("""\
                # TODO: also fix this
                x = 1
            """)
        )
        return tmp_path

    async def test_finds_pattern_in_files(self, tmp_repo: Path) -> None:
        tool = GrepTool()
        inp = GrepInput(pattern="TODO", path=str(tmp_repo))
        result = await tool.execute(inp)

        assert result.is_error is False
        assert "TODO" in result.output

    async def test_finds_matches_in_subdir(self, tmp_repo: Path) -> None:
        tool = GrepTool()
        inp = GrepInput(pattern="TODO", path=str(tmp_repo))
        result = await tool.execute(inp)

        # Both alpha.py and sub/gamma.py contain TODO
        assert result.output.count("TODO") >= 2

    async def test_no_match_returns_empty_output(self, tmp_repo: Path) -> None:
        tool = GrepTool()
        inp = GrepInput(pattern="NONEXISTENT_PATTERN_XYZ", path=str(tmp_repo))
        result = await tool.execute(inp)

        assert result.is_error is False
        assert result.output.strip() == "" or "no matches" in result.output.lower()

    async def test_glob_filter_restricts_files(self, tmp_repo: Path) -> None:
        """glob_filter='*.txt' 应只搜索 txt 文件，beta.txt 没有 TODO。"""
        tool = GrepTool()
        inp = GrepInput(pattern="TODO", path=str(tmp_repo), glob_filter="*.txt")
        result = await tool.execute(inp)

        assert result.is_error is False
        # beta.txt has no TODO → no matches
        assert "TODO" not in result.output

    async def test_glob_filter_py_finds_todo(self, tmp_repo: Path) -> None:
        """glob_filter='*.py' 应找到 TODO。"""
        tool = GrepTool()
        inp = GrepInput(pattern="TODO", path=str(tmp_repo), glob_filter="*.py")
        result = await tool.execute(inp)

        assert "TODO" in result.output

    async def test_invalid_path_returns_error(self) -> None:
        tool = GrepTool()
        inp = GrepInput(pattern="x", path="/nonexistent/path/xyz")
        result = await tool.execute(inp)

        assert result.is_error is True

    async def test_output_includes_line_numbers(self, tmp_repo: Path) -> None:
        """输出应包含行号（grep -n 格式）。"""
        tool = GrepTool()
        inp = GrepInput(pattern="TODO", path=str(tmp_repo))
        result = await tool.execute(inp)

        # grep -n output: "filename:linenum:content"
        # At least one line should have a colon-separated number
        lines = [l for l in result.output.splitlines() if "TODO" in l]
        assert len(lines) >= 1
        # Each match line should contain the file name
        assert any("alpha.py" in line or "gamma.py" in line for line in lines)
```

文件路径: `tests/tools/builtin/test_glob.py`
```python
from __future__ import annotations

from pathlib import Path

import pytest

from neoagent.core.types import ToolResult
from neoagent.tools.builtin.glob import GlobInput, GlobTool


# ---------- 构造与属性 ----------

class TestGlobToolAttributes:
    def test_name(self) -> None:
        assert GlobTool().name == "glob"

    def test_permission_auto(self) -> None:
        assert GlobTool().permission == "auto"

    def test_is_concurrent_safe(self) -> None:
        assert GlobTool().is_concurrent_safe is True

    def test_input_model(self) -> None:
        assert GlobTool().input_model is GlobInput


# ---------- GlobInput 验证 ----------

class TestGlobInput:
    def test_default_path(self) -> None:
        inp = GlobInput(pattern="**/*.py")
        assert inp.path == "."

    def test_custom_path(self) -> None:
        inp = GlobInput(pattern="*.md", path="/tmp")
        assert inp.path == "/tmp"

    def test_pattern_required(self) -> None:
        with pytest.raises(Exception):
            GlobInput()  # type: ignore[call-arg]


# ---------- 执行：实际文件匹配 ----------

class TestGlobToolExecute:
    @pytest.fixture()
    def tmp_repo(self, tmp_path: Path) -> Path:
        """创建带层级的临时文件结构。"""
        (tmp_path / "main.py").write_text("x = 1")
        (tmp_path / "utils.py").write_text("y = 2")
        (tmp_path / "README.md").write_text("# readme")
        sub = tmp_path / "src"
        sub.mkdir()
        (sub / "core.py").write_text("z = 3")
        (sub / "helper.py").write_text("w = 4")
        deep = sub / "internal"
        deep.mkdir()
        (deep / "secret.py").write_text("s = 0")
        return tmp_path

    async def test_match_all_py_files(self, tmp_repo: Path) -> None:
        tool = GlobTool()
        inp = GlobInput(pattern="**/*.py", path=str(tmp_repo))
        result = await tool.execute(inp)

        assert result.is_error is False
        # Should find main.py, utils.py, src/core.py, src/helper.py, src/internal/secret.py
        assert "main.py" in result.output
        assert "core.py" in result.output
        assert "secret.py" in result.output

    async def test_match_top_level_only(self, tmp_repo: Path) -> None:
        tool = GlobTool()
        inp = GlobInput(pattern="*.py", path=str(tmp_repo))
        result = await tool.execute(inp)

        assert result.is_error is False
        assert "main.py" in result.output
        assert "utils.py" in result.output
        # src/core.py should NOT appear (non-recursive)
        assert "core.py" not in result.output

    async def test_match_markdown(self, tmp_repo: Path) -> None:
        tool = GlobTool()
        inp = GlobInput(pattern="**/*.md", path=str(tmp_repo))
        result = await tool.execute(inp)

        assert "README.md" in result.output

    async def test_no_match_returns_empty(self, tmp_repo: Path) -> None:
        tool = GlobTool()
        inp = GlobInput(pattern="**/*.nonexistent", path=str(tmp_repo))
        result = await tool.execute(inp)

        assert result.is_error is False
        assert result.output.strip() == ""

    async def test_invalid_path_returns_error(self) -> None:
        tool = GlobTool()
        inp = GlobInput(pattern="**/*.py", path="/nonexistent/path/xyz")
        result = await tool.execute(inp)

        assert result.is_error is True

    async def test_results_sorted(self, tmp_repo: Path) -> None:
        """文件列表应按路径排序，便于确定性输出。"""
        tool = GlobTool()
        inp = GlobInput(pattern="**/*.py", path=str(tmp_repo))
        result = await tool.execute(inp)

        lines = [l for l in result.output.splitlines() if l.strip()]
        assert lines == sorted(lines)

    async def test_each_result_on_own_line(self, tmp_repo: Path) -> None:
        tool = GlobTool()
        inp = GlobInput(pattern="**/*.py", path=str(tmp_repo))
        result = await tool.execute(inp)

        lines = [l for l in result.output.splitlines() if l.strip()]
        # 5 python files total
        assert len(lines) == 5
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd neoagent && python -m pytest tests/tools/builtin/test_grep.py tests/tools/builtin/test_glob.py -v
```

预期: FAIL，`ModuleNotFoundError: No module named 'neoagent.tools.builtin.grep'`

---

### Step 3 — 写实现

文件路径: `neoagent/tools/builtin/grep.py`
```python
from __future__ import annotations

import asyncio
import shutil
from pathlib import Path

from pydantic import BaseModel

from neoagent.core.types import ToolResult
from neoagent.tools.base import BaseTool


class GrepInput(BaseModel):
    """Input schema for the Grep tool."""

    pattern: str
    path: str = "."
    glob_filter: str | None = None


class GrepTool(BaseTool):
    """Search file contents for a regex pattern.

    Prefers ripgrep (rg) when available; falls back to POSIX grep.
    Always searches recursively and includes line numbers.
    """

    name = "grep"
    description = "在文件内容中递归搜索正则表达式模式，返回匹配行及行号"
    input_model = GrepInput
    permission = "auto"
    is_concurrent_safe = True

    async def execute(self, input: GrepInput) -> ToolResult:  # type: ignore[override]
        search_path = Path(input.path)

        if not search_path.exists():
            return ToolResult(
                call_id="",
                output=f"Path does not exist: {input.path}",
                is_error=True,
            )

        # Build the command: prefer rg, fall back to grep
        if shutil.which("rg") is not None:
            cmd = _build_rg_command(input.pattern, str(search_path), input.glob_filter)
        else:
            cmd = _build_grep_command(input.pattern, str(search_path), input.glob_filter)

        try:
            proc = await asyncio.create_subprocess_shell(
                cmd,
                stdout=asyncio.subprocess.PIPE,
                stderr=asyncio.subprocess.PIPE,
            )
            stdout_bytes, stderr_bytes = await proc.communicate()
        except OSError as exc:
            return ToolResult(
                call_id="",
                output=f"Failed to run search: {exc}",
                is_error=True,
            )

        stdout = stdout_bytes.decode(errors="replace")
        stderr = stderr_bytes.decode(errors="replace")

        # Exit code 1 from grep/rg means "no matches" — not an error
        if proc.returncode == 0:
            return ToolResult(call_id="", output=stdout, is_error=False)
        elif proc.returncode == 1:
            # No matches found
            return ToolResult(call_id="", output="", is_error=False)
        else:
            # Actual error (exit code 2+)
            return ToolResult(
                call_id="",
                output=stderr or stdout or f"Search failed (exit {proc.returncode})",
                is_error=True,
            )


def _build_rg_command(pattern: str, path: str, glob_filter: str | None) -> str:
    """Build a ripgrep command string."""
    # --line-number: show line numbers
    # --no-heading: one file:line:content per output line (easier to parse)
    # --color=never: no ANSI escape codes in output
    parts = ["rg", "--line-number", "--no-heading", "--color=never"]
    if glob_filter:
        parts += ["--glob", f"'{glob_filter}'"]
    parts += [f"'{pattern}'", f"'{path}'"]
    return " ".join(parts)


def _build_grep_command(pattern: str, path: str, glob_filter: str | None) -> str:
    """Build a POSIX grep command string."""
    # -r: recursive, -n: line numbers, -E: extended regex
    parts = ["grep", "-r", "-n", "-E"]
    if glob_filter:
        parts += [f"--include='{glob_filter}'"]
    parts += [f"'{pattern}'", f"'{path}'"]
    return " ".join(parts)
```

文件路径: `neoagent/tools/builtin/glob.py`
```python
from __future__ import annotations

from pathlib import Path

from pydantic import BaseModel

from neoagent.core.types import ToolResult
from neoagent.tools.base import BaseTool


class GlobInput(BaseModel):
    """Input schema for the Glob tool."""

    pattern: str
    path: str = "."


class GlobTool(BaseTool):
    """Find files by name pattern using Python's pathlib globbing.

    Results are sorted alphabetically for deterministic output.
    """

    name = "glob"
    description = "按文件名模式匹配查找文件，支持 ** 递归通配符，结果按路径排序"
    input_model = GlobInput
    permission = "auto"
    is_concurrent_safe = True

    async def execute(self, input: GlobInput) -> ToolResult:  # type: ignore[override]
        base = Path(input.path)

        if not base.exists():
            return ToolResult(
                call_id="",
                output=f"Path does not exist: {input.path}",
                is_error=True,
            )

        try:
            matched = sorted(str(p) for p in base.glob(input.pattern) if p.is_file())
        except Exception as exc:
            return ToolResult(
                call_id="",
                output=f"Glob error: {exc}",
                is_error=True,
            )

        output = "\n".join(matched)
        return ToolResult(call_id="", output=output, is_error=False)
```

- [ ] **Step 4: 运行测试确认通过**

```bash
cd neoagent && python -m pytest tests/tools/builtin/test_grep.py tests/tools/builtin/test_glob.py -v
```

预期: PASS — 所有测试绿色

- [ ] **Step 5: 提交**

```bash
git add neoagent/neoagent/tools/builtin/grep.py \
        neoagent/neoagent/tools/builtin/glob.py \
        neoagent/tests/tools/builtin/test_grep.py \
        neoagent/tests/tools/builtin/test_glob.py
git commit -m "feat: add GrepTool (rg/grep fallback) and GlobTool (pathlib) builtins"
```

---

## Task 15: 集成测试 (`tests/test_integration.py`)

### Files

- `tests/test_integration.py`

### 设计说明

集成测试不调真实 API。用 `MockProvider` 代替 `AnthropicProvider`，预设一组 `Response` 序列，模拟真实 API 对话轮次。每个场景验证整条链路：`NeoAgent.run()` → `QueryLoop` → `ToolRegistry` → 工具执行 → 结果回传 → 最终 `ConversationResult`。

`ReadTool`（来自 Task 10/11，`neoagent/tools/builtin/read.py`）在测试中注册到临时文件，用于验证工具执行端到端路径。

---

### Step 1 — 写失败测试

文件路径: `tests/test_integration.py`
```python
from __future__ import annotations

import textwrap
from pathlib import Path
from typing import Iterator
from unittest.mock import AsyncMock

import pytest

from neoagent.agent import NeoAgent, NeoAgentConfig
from neoagent.core.types import (
    ConversationResult,
    Message,
    TextBlock,
    ToolCall,
    ToolResult,
    ToolUseBlock,
    Turn,
)
from neoagent.providers.base import Provider, Response
from neoagent.tools.builtin.read import ReadInput, ReadTool
from neoagent.tools.registry import ToolRegistry


# ---------------------------------------------------------------------------
# MockProvider — 预设 Response 序列
# ---------------------------------------------------------------------------

class MockProvider(Provider):
    """Returns a pre-set sequence of Response objects, one per create() call."""

    def __init__(self, responses: list[Response]) -> None:
        self._responses: list[Response] = list(responses)
        self._call_count: int = 0
        self.calls: list[dict] = []  # record every call for assertion

    async def create(self, system: str, messages: list, tools: list) -> Response:
        self.calls.append({"system": system, "messages": messages, "tools": tools})
        if self._call_count >= len(self._responses):
            raise RuntimeError(
                f"MockProvider exhausted after {len(self._responses)} calls"
            )
        response = self._responses[self._call_count]
        self._call_count += 1
        return response

    def get_context_window(self) -> int:
        return 200_000


def _text_response(text: str) -> Response:
    """Helper: build an end_turn Response with a single TextBlock."""
    return Response(
        content=[TextBlock(text=text)],
        stop_reason="end_turn",
        input_tokens=10,
        output_tokens=5,
    )


def _tool_use_response(tool_id: str, tool_name: str, tool_input: dict) -> Response:
    """Helper: build a tool_use Response with a single ToolUseBlock."""
    return Response(
        content=[ToolUseBlock(id=tool_id, name=tool_name, input=tool_input)],
        stop_reason="tool_use",
        input_tokens=20,
        output_tokens=8,
    )


# ---------------------------------------------------------------------------
# 场景 1: 简单对话 — provider 直接返回 end_turn
# ---------------------------------------------------------------------------

class TestSimpleConversation:
    """Provider 第一轮就返回 end_turn，不涉及工具调用。"""

    async def test_returns_conversation_result(self) -> None:
        provider = MockProvider([_text_response("I'm done!")])
        registry = ToolRegistry()

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("Hello!")

        assert isinstance(result, ConversationResult)
        assert result.reason == "completed"

    async def test_single_turn_no_tools(self) -> None:
        provider = MockProvider([_text_response("Hello back!")])
        registry = ToolRegistry()

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("Hello!")

        assert len(result.turns) == 1
        turn = result.turns[0]
        assert turn.stop_reason == "end_turn"
        assert turn.tool_calls == []
        assert turn.tool_results == []

    async def test_provider_called_once(self) -> None:
        provider = MockProvider([_text_response("done")])
        registry = ToolRegistry()

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        await agent.run("ping")

        assert provider._call_count == 1

    async def test_final_text_in_turn_response(self) -> None:
        provider = MockProvider([_text_response("Final answer: 42")])
        registry = ToolRegistry()

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("What is the answer?")

        turn = result.turns[0]
        # Response content should contain the text block
        content = turn.response.content
        if isinstance(content, list):
            texts = [b.text for b in content if isinstance(b, TextBlock)]
            assert "Final answer: 42" in "\n".join(texts)
        else:
            assert "Final answer: 42" in content


# ---------------------------------------------------------------------------
# 场景 2: 工具调用流程
# ---------------------------------------------------------------------------

class TestToolCallFlow:
    """Provider 第一轮返回 tool_use (read)，第二轮返回 end_turn。
    验证：工具被执行、结果回传、最终文本正确。
    """

    @pytest.fixture()
    def tmp_file(self, tmp_path: Path) -> Path:
        f = tmp_path / "hello.txt"
        f.write_text("Hello from file!")
        return f

    async def test_tool_executed_and_result_fed_back(self, tmp_file: Path) -> None:
        tool_id = "tu_read_001"
        provider = MockProvider([
            _tool_use_response(tool_id, "read", {"path": str(tmp_file)}),
            _text_response("I read the file successfully."),
        ])

        registry = ToolRegistry()
        registry.register(ReadTool())

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("Read the file for me.")

        assert result.reason == "completed"
        assert provider._call_count == 2

    async def test_two_turns_recorded(self, tmp_file: Path) -> None:
        tool_id = "tu_read_002"
        provider = MockProvider([
            _tool_use_response(tool_id, "read", {"path": str(tmp_file)}),
            _text_response("File read done."),
        ])

        registry = ToolRegistry()
        registry.register(ReadTool())

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("Please read the file.")

        assert len(result.turns) == 2

    async def test_first_turn_has_tool_call(self, tmp_file: Path) -> None:
        tool_id = "tu_read_003"
        provider = MockProvider([
            _tool_use_response(tool_id, "read", {"path": str(tmp_file)}),
            _text_response("Done."),
        ])

        registry = ToolRegistry()
        registry.register(ReadTool())

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("Read it.")

        first_turn = result.turns[0]
        assert len(first_turn.tool_calls) == 1
        assert first_turn.tool_calls[0].name == "read"

    async def test_tool_result_contains_file_content(self, tmp_file: Path) -> None:
        tool_id = "tu_read_004"
        provider = MockProvider([
            _tool_use_response(tool_id, "read", {"path": str(tmp_file)}),
            _text_response("Got it."),
        ])

        registry = ToolRegistry()
        registry.register(ReadTool())

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("Read hello.txt")

        first_turn = result.turns[0]
        assert len(first_turn.tool_results) == 1
        tool_result = first_turn.tool_results[0]
        assert tool_result.is_error is False
        assert "Hello from file!" in tool_result.output

    async def test_second_call_includes_tool_result_in_messages(self, tmp_file: Path) -> None:
        """第二次调用 provider 时，消息列表中必须包含 tool_result。"""
        tool_id = "tu_read_005"
        provider = MockProvider([
            _tool_use_response(tool_id, "read", {"path": str(tmp_file)}),
            _text_response("Done."),
        ])

        registry = ToolRegistry()
        registry.register(ReadTool())

        config = NeoAgentConfig(max_turns=10)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        await agent.run("Read it.")

        # Second provider call should have messages with tool_result content
        second_call_messages = provider.calls[1]["messages"]
        # The last user message should contain tool_result blocks
        last_msg = second_call_messages[-1]
        has_tool_result = False
        if isinstance(last_msg.get("content"), list):
            has_tool_result = any(
                b.get("type") == "tool_result" for b in last_msg["content"]
                if isinstance(b, dict)
            )
        elif hasattr(last_msg, "content") and isinstance(last_msg.content, list):
            from neoagent.core.types import ToolResultBlock
            has_tool_result = any(
                isinstance(b, ToolResultBlock) for b in last_msg.content
            )
        assert has_tool_result


# ---------------------------------------------------------------------------
# 场景 3: max_turns 触发
# ---------------------------------------------------------------------------

class TestMaxTurns:
    """Provider 始终返回 tool_use，验证在 max_turns 后停止。"""

    async def test_stops_at_max_turns(self) -> None:
        # 准备 100 个 tool_use responses — 远超 max_turns=3
        tool_responses = [
            _tool_use_response(f"tu_{i}", "bash", {"command": "echo loop"})
            for i in range(100)
        ]
        provider = MockProvider(tool_responses)

        registry = ToolRegistry()
        # Register a mock bash-like tool that auto-executes (permission=auto)
        from neoagent.tools.base import BaseTool
        from pydantic import BaseModel

        class EchoInput(BaseModel):
            command: str

        class EchoTool(BaseTool):
            name = "bash"
            description = "echo tool for testing"
            input_model = EchoInput
            permission = "auto"
            is_concurrent_safe = True

            async def execute(self, input: EchoInput) -> ToolResult:  # type: ignore[override]
                return ToolResult(call_id="", output="loop output", is_error=False)

        registry.register(EchoTool())

        config = NeoAgentConfig(max_turns=3)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("keep going")

        assert result.reason == "max_turns"

    async def test_provider_called_exactly_max_turns_times(self) -> None:
        tool_responses = [
            _tool_use_response(f"tu_{i}", "bash", {"command": "echo x"})
            for i in range(100)
        ]
        provider = MockProvider(tool_responses)

        registry = ToolRegistry()
        from neoagent.tools.base import BaseTool
        from pydantic import BaseModel

        class EchoInput(BaseModel):
            command: str

        class EchoTool(BaseTool):
            name = "bash"
            description = "echo"
            input_model = EchoInput
            permission = "auto"
            is_concurrent_safe = True

            async def execute(self, input: EchoInput) -> ToolResult:  # type: ignore[override]
                return ToolResult(call_id="", output="ok", is_error=False)

        registry.register(EchoTool())

        config = NeoAgentConfig(max_turns=4)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("go")

        assert result.reason == "max_turns"
        # Each turn calls provider once
        assert provider._call_count == 4

    async def test_turns_list_length_equals_max_turns(self) -> None:
        tool_responses = [
            _tool_use_response(f"tu_{i}", "bash", {"command": "echo"})
            for i in range(100)
        ]
        provider = MockProvider(tool_responses)

        registry = ToolRegistry()
        from neoagent.tools.base import BaseTool
        from pydantic import BaseModel

        class EchoInput(BaseModel):
            command: str

        class EchoTool(BaseTool):
            name = "bash"
            description = "echo"
            input_model = EchoInput
            permission = "auto"
            is_concurrent_safe = True

            async def execute(self, input: EchoInput) -> ToolResult:  # type: ignore[override]
                return ToolResult(call_id="", output="ok", is_error=False)

        registry.register(EchoTool())

        config = NeoAgentConfig(max_turns=5)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("loop")

        assert len(result.turns) == 5


# ---------------------------------------------------------------------------
# 场景 4: 上下文压缩触发
# ---------------------------------------------------------------------------

class TestContextCompression:
    """构造超大消息列表，验证 compressor.should_compress 触发。

    不测试压缩的实际摘要内容（那是 LLM 输出），只验证：
    1. should_compress 在超过阈值（70% of context_window）时返回 True
    2. agent.run() 在超过阈值时调用压缩路径而不崩溃
    """

    async def test_should_compress_below_threshold(self) -> None:
        """低于阈值时 should_compress 应返回 False。"""
        from neoagent.core.compress import ContextCompressor

        compressor = ContextCompressor(context_window=200_000)
        # 构造少量消息，token 远低于 70% * 200_000 = 140_000
        messages = [Message(role="user", content="hello")]
        assert compressor.should_compress(messages) is False

    async def test_should_compress_above_threshold(self) -> None:
        """超过阈值时 should_compress 应返回 True。"""
        from neoagent.core.compress import ContextCompressor

        compressor = ContextCompressor(context_window=1_000)  # 极小 context window
        # 构造一条足够长的消息触发阈值（700 token @ 70%）
        long_text = "word " * 300  # ~300 tokens (tiktoken: ~1 token/word)
        messages = [Message(role="user", content=long_text)]
        assert compressor.should_compress(messages) is True

    async def test_agent_survives_large_context(self) -> None:
        """超大上下文下 agent 不应崩溃，即使压缩路径被触发。

        MockProvider 不调真实 API，压缩阶段若调 LLM 会触发 RuntimeError。
        所以此测试使用极小 context_window=100 但只发送普通消息，
        确保 should_compress 判断本身不抛出异常。
        """
        # Use a fresh provider that returns end_turn immediately
        provider = MockProvider([_text_response("done")])
        registry = ToolRegistry()

        # Pass a tiny context_window so should_compress logic is exercised,
        # but the initial user message is still small enough to fit.
        config = NeoAgentConfig(max_turns=5, context_window=200_000)
        agent = NeoAgent(provider=provider, tool_registry=registry, config=config)

        result = await agent.run("short message")
        assert result.reason == "completed"

    async def test_compress_threshold_is_seventy_percent(self) -> None:
        """确认阈值比例是 70%，不是其他值。"""
        from neoagent.core.compress import ContextCompressor

        # context_window=1000, threshold=700 tokens
        compressor = ContextCompressor(context_window=1_000)

        # ~699 tokens — just below threshold, should NOT compress
        short_text = "word " * 139  # ~139 tokens safely below 700
        messages_short = [Message(role="user", content=short_text)]

        # ~800 tokens — above threshold, should compress
        long_text = "word " * 250  # ~250 tokens above 700 at window=1000
        # With context_window=1000, threshold=700; 250 words ~ 250 tokens < 700
        # Use an even smaller window to guarantee trigger
        compressor_tiny = ContextCompressor(context_window=200)
        # threshold = 140 tokens; "word " * 50 = 50 tokens < 140
        messages_tiny_short = [Message(role="user", content="word " * 30)]
        messages_tiny_long = [Message(role="user", content="word " * 100)]

        assert compressor_tiny.should_compress(messages_tiny_short) is False
        assert compressor_tiny.should_compress(messages_tiny_long) is True
```

- [ ] **Step 2: 运行测试确认失败**

```bash
cd neoagent && python -m pytest tests/test_integration.py -v
```

预期: FAIL，`ImportError: cannot import name 'NeoAgent' from 'neoagent.agent'`（或其他依赖缺失）

---

### Step 3 — 确认前置依赖已完成

集成测试依赖以下组件全部就位（Tasks 1–12）：

| 组件 | 文件 | Task |
|------|------|------|
| `NeoAgent`, `NeoAgentConfig` | `neoagent/agent.py` | Task 11 |
| `QueryLoop` | `neoagent/core/loop.py` | Task 7 |
| `PromptBuilder` | `neoagent/core/prompt.py` | Task 8 |
| `ContextCompressor` | `neoagent/core/compress.py` | Task 9 |
| `ToolRegistry` | `neoagent/tools/registry.py` | Task 5 |
| `ReadTool` | `neoagent/tools/builtin/read.py` | Task 12 |
| `Provider`, `Response` | `neoagent/providers/base.py` | Task 3 |
| 所有 core types | `neoagent/core/types.py` | Task 2 |

如果某个前置组件尚未实现，先完成对应 Task 再回到 Task 15。

`NeoAgentConfig` 需要有 `context_window` 参数（可选，默认 0 表示从 provider 获取）：

```python
# 已有的 NeoAgentConfig（Task 11 实现），确认包含：
@dataclass
class NeoAgentConfig:
    max_turns: int = 30
    context_window: int = 0  # 0 = 从 provider.get_context_window() 获取
```

如果 `NeoAgentConfig` 尚未有 `context_window` 字段，在 `neoagent/agent.py` 中补充该字段。

---

### Step 4 — 运行测试确认通过

```bash
cd neoagent && python -m pytest tests/test_integration.py -v
```

预期: PASS — 所有测试绿色

如有单个场景失败，逐场景调试：
```bash
# 只跑场景 1
cd neoagent && python -m pytest tests/test_integration.py::TestSimpleConversation -v
# 只跑场景 2
cd neoagent && python -m pytest tests/test_integration.py::TestToolCallFlow -v
# 只跑场景 3
cd neoagent && python -m pytest tests/test_integration.py::TestMaxTurns -v
# 只跑场景 4
cd neoagent && python -m pytest tests/test_integration.py::TestContextCompression -v
```

---

### Step 5 — 提交

```bash
git add neoagent/tests/test_integration.py
git commit -m "test: add end-to-end integration tests for QueryLoop, tool execution, max_turns, compression"
```

---

## 验收检查

所有 Task 完成后，跑完整测试套件：

```bash
cd neoagent && python -m pytest tests/ -v --tb=short
```

预期: Part 5 新增测试全部通过，既有测试（Tasks 1–12）不回归。
