# neoagent v1 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 实现 neoagent v1 极简核 — Python agent 框架，包含 Query Loop、Tool System、Prompt System 和基础上下文压缩

**Architecture:** Claude Code 骨架 + 各家优点 + 自研改进。简单 while 循环驱动 agent loop，BaseTool + Pydantic 工具系统，Section-based prompt 动态组装。以 Anthropic Claude 为唯一 provider。

**Tech Stack:** Python 3.11+, anthropic SDK, pydantic v2, tiktoken, pytest + pytest-asyncio

---

### Task 1: 项目脚手架

**Files:**
- Create: `neoagent/pyproject.toml`
- Create: `neoagent/neoagent/__init__.py`
- Create: `neoagent/neoagent/core/__init__.py`
- Create: `neoagent/neoagent/tools/__init__.py`
- Create: `neoagent/neoagent/tools/builtin/__init__.py`
- Create: `neoagent/neoagent/providers/__init__.py`
- Create: `neoagent/tests/__init__.py`
- Create: `neoagent/tests/test_scaffold.py`

- [ ] **Step 1: Write failing test**

`neoagent/tests/test_scaffold.py`
```python
from __future__ import annotations


def test_package_importable() -> None:
    import neoagent  # noqa: F401
    assert True


def test_core_importable() -> None:
    import neoagent.core  # noqa: F401
    assert True


def test_tools_importable() -> None:
    import neoagent.tools  # noqa: F401
    assert True


def test_providers_importable() -> None:
    import neoagent.providers  # noqa: F401
    assert True
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/test_scaffold.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'neoagent'`

- [ ] **Step 3: Write implementation**

`neoagent/pyproject.toml`
```toml
[build-system]
requires = ["setuptools>=68", "wheel"]
build-backend = "setuptools.backends.legacy:build"

[project]
name = "neoagent"
version = "0.1.0"
description = "Minimal Python agent framework — Query Loop + Tool System + Prompt System"
requires-python = ">=3.11"
dependencies = [
    "anthropic>=0.28.0",
    "pydantic>=2.0",
    "tiktoken>=0.7.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
]

[tool.setuptools.packages.find]
where = ["."]
include = ["neoagent*"]

[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]
```

`neoagent/neoagent/__init__.py`
```python
from __future__ import annotations

__version__ = "0.1.0"
```

`neoagent/neoagent/core/__init__.py`
```python
from __future__ import annotations
```

`neoagent/neoagent/tools/__init__.py`
```python
from __future__ import annotations
```

`neoagent/neoagent/tools/builtin/__init__.py`
```python
from __future__ import annotations
```

`neoagent/neoagent/providers/__init__.py`
```python
from __future__ import annotations
```

`neoagent/tests/__init__.py`
```python
from __future__ import annotations
```

Then install in editable mode:
```bash
cd neoagent && pip install -e ".[dev]"
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/test_scaffold.py -v`
Expected: PASS — 4 tests collected, all green

- [ ] **Step 5: Commit**

```bash
git add neoagent/pyproject.toml \
        neoagent/neoagent/__init__.py \
        neoagent/neoagent/core/__init__.py \
        neoagent/neoagent/tools/__init__.py \
        neoagent/neoagent/tools/builtin/__init__.py \
        neoagent/neoagent/providers/__init__.py \
        neoagent/tests/__init__.py \
        neoagent/tests/test_scaffold.py
git commit -m "feat: scaffold neoagent project structure with pyproject.toml"
```

---

### Task 2: 核心类型 (neoagent/core/types.py)

**Files:**
- Create: `neoagent/neoagent/core/types.py`
- Test: `neoagent/tests/core/test_types.py`

- [ ] **Step 1: Write failing test**

`neoagent/tests/core/__init__.py`
```python
from __future__ import annotations
```

`neoagent/tests/core/test_types.py`
```python
from __future__ import annotations

import pytest
from neoagent.core.types import (
    ConversationResult,
    Message,
    TextBlock,
    ToolCall,
    ToolResult,
    ToolResultBlock,
    ToolUseBlock,
    Turn,
)


# --- ContentBlock tests ---

class TestTextBlock:
    def test_construction(self) -> None:
        block = TextBlock(text="hello")
        assert block.type == "text"
        assert block.text == "hello"

    def test_type_is_literal(self) -> None:
        block = TextBlock(text="x")
        assert block.type == "text"


class TestToolUseBlock:
    def test_construction(self) -> None:
        block = ToolUseBlock(id="tu_1", name="read", input={"path": "/tmp/f"})
        assert block.type == "tool_use"
        assert block.id == "tu_1"
        assert block.name == "read"
        assert block.input == {"path": "/tmp/f"}


class TestToolResultBlock:
    def test_construction_success(self) -> None:
        block = ToolResultBlock(tool_use_id="tu_1", content="file contents")
        assert block.type == "tool_result"
        assert block.tool_use_id == "tu_1"
        assert block.content == "file contents"
        assert block.is_error is False

    def test_construction_error(self) -> None:
        block = ToolResultBlock(tool_use_id="tu_1", content="not found", is_error=True)
        assert block.is_error is True


# --- Message tests ---

class TestMessage:
    def test_string_content(self) -> None:
        msg = Message(role="user", content="hello")
        assert msg.role == "user"
        assert msg.content == "hello"

    def test_block_content(self) -> None:
        blocks = [TextBlock(text="hi"), ToolUseBlock(id="tu_1", name="bash", input={})]
        msg = Message(role="assistant", content=blocks)
        assert isinstance(msg.content, list)
        assert len(msg.content) == 2

    def test_role_is_validated(self) -> None:
        with pytest.raises(Exception):
            Message(role="system", content="oops")  # type: ignore[arg-type]


# --- ToolCall tests ---

class TestToolCall:
    def test_construction(self) -> None:
        call = ToolCall(id="tc_1", name="grep", input={"pattern": "TODO"})
        assert call.id == "tc_1"
        assert call.name == "grep"
        assert call.input == {"pattern": "TODO"}


# --- ToolResult tests ---

class TestToolResult:
    def test_construction_default(self) -> None:
        result = ToolResult(call_id="tc_1", output="found 3 matches")
        assert result.call_id == "tc_1"
        assert result.output == "found 3 matches"
        assert result.is_error is False

    def test_construction_error(self) -> None:
        result = ToolResult(call_id="tc_1", output="permission denied", is_error=True)
        assert result.is_error is True


# --- Turn tests ---

class TestTurn:
    def test_construction_end_turn(self) -> None:
        msg = Message(role="assistant", content="done")
        turn = Turn(
            response=msg,
            tool_calls=[],
            tool_results=[],
            stop_reason="end_turn",
        )
        assert turn.stop_reason == "end_turn"
        assert turn.tool_calls == []
        assert turn.tool_results == []

    def test_construction_tool_use(self) -> None:
        msg = Message(role="assistant", content=[])
        call = ToolCall(id="tc_1", name="read", input={"path": "/f"})
        result = ToolResult(call_id="tc_1", output="content")
        turn = Turn(
            response=msg,
            tool_calls=[call],
            tool_results=[result],
            stop_reason="tool_use",
        )
        assert turn.stop_reason == "tool_use"
        assert len(turn.tool_calls) == 1
        assert len(turn.tool_results) == 1

    def test_stop_reason_validated(self) -> None:
        msg = Message(role="assistant", content="x")
        with pytest.raises(Exception):
            Turn(
                response=msg,
                tool_calls=[],
                tool_results=[],
                stop_reason="unknown",  # type: ignore[arg-type]
            )


# --- ConversationResult tests ---

class TestConversationResult:
    def test_completed(self) -> None:
        result = ConversationResult(turns=[], reason="completed")
        assert result.reason == "completed"
        assert result.turns == []

    def test_max_turns(self) -> None:
        result = ConversationResult(turns=[], reason="max_turns")
        assert result.reason == "max_turns"

    def test_reason_validated(self) -> None:
        with pytest.raises(Exception):
            ConversationResult(turns=[], reason="timeout")  # type: ignore[arg-type]
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/core/test_types.py -v`
Expected: FAIL with `ImportError: cannot import name 'TextBlock' from 'neoagent.core.types'`

- [ ] **Step 3: Write implementation**

`neoagent/neoagent/core/types.py`
```python
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Literal

from pydantic import BaseModel


# ---------------------------------------------------------------------------
# Content blocks
# ---------------------------------------------------------------------------

class TextBlock(BaseModel):
    """Plain text content returned by the model."""

    type: Literal["text"] = "text"
    text: str


class ToolUseBlock(BaseModel):
    """A tool invocation requested by the model."""

    type: Literal["tool_use"] = "tool_use"
    id: str
    name: str
    input: dict


class ToolResultBlock(BaseModel):
    """The result of executing a tool, to be fed back to the model."""

    type: Literal["tool_result"] = "tool_result"
    tool_use_id: str
    content: str
    is_error: bool = False


ContentBlock = TextBlock | ToolUseBlock | ToolResultBlock


# ---------------------------------------------------------------------------
# Message
# ---------------------------------------------------------------------------

class Message(BaseModel):
    """A single message in the conversation history."""

    role: Literal["user", "assistant"]
    content: str | list[ContentBlock]


# ---------------------------------------------------------------------------
# Tool primitives
# ---------------------------------------------------------------------------

@dataclass
class ToolCall:
    """A tool call extracted from the model's response."""

    id: str
    name: str
    input: dict


@dataclass
class ToolResult:
    """The result of executing a single tool call."""

    call_id: str
    output: str
    is_error: bool = False


# ---------------------------------------------------------------------------
# Turn & ConversationResult
# ---------------------------------------------------------------------------

class Turn(BaseModel):
    """One full agentic turn: model response + tool execution results."""

    response: Message
    tool_calls: list[ToolCall] = field(default_factory=list)
    tool_results: list[ToolResult] = field(default_factory=list)
    stop_reason: Literal["end_turn", "tool_use", "max_tokens"]

    model_config = {"arbitrary_types_allowed": True}


class ConversationResult(BaseModel):
    """The final result of a completed (or truncated) agent run."""

    turns: list[Turn]
    reason: Literal["completed", "max_turns"]

    model_config = {"arbitrary_types_allowed": True}
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/core/test_types.py -v`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add neoagent/neoagent/core/types.py \
        neoagent/tests/core/__init__.py \
        neoagent/tests/core/test_types.py
git commit -m "feat: add core types — Message, Turn, ToolCall, ToolResult, ConversationResult"
```

---

### Task 3: Provider 抽象 + Anthropic 实现 (neoagent/providers/)

**Files:**
- Create: `neoagent/neoagent/providers/base.py`
- Create: `neoagent/neoagent/providers/anthropic.py`
- Test: `neoagent/tests/providers/test_anthropic_provider.py`

- [ ] **Step 1: Write failing test**

`neoagent/tests/providers/__init__.py`
```python
from __future__ import annotations
```

`neoagent/tests/providers/test_anthropic_provider.py`
```python
from __future__ import annotations

from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from neoagent.core.types import Message, TextBlock, ToolUseBlock
from neoagent.providers.anthropic import AnthropicProvider
from neoagent.providers.base import Provider, Response


# ---------------------------------------------------------------------------
# Provider ABC
# ---------------------------------------------------------------------------

class TestProviderABC:
    def test_is_abstract(self) -> None:
        """Provider cannot be instantiated directly."""
        with pytest.raises(TypeError):
            Provider()  # type: ignore[abstract]

    def test_concrete_must_implement_create(self) -> None:
        """Subclass without create() raises TypeError."""
        class BadProvider(Provider):
            def get_context_window(self) -> int:
                return 200_000

        with pytest.raises(TypeError):
            BadProvider()  # type: ignore[abstract]

    def test_concrete_must_implement_get_context_window(self) -> None:
        class BadProvider(Provider):
            async def create(self, system, messages, tools) -> Response:
                ...

        with pytest.raises(TypeError):
            BadProvider()  # type: ignore[abstract]


# ---------------------------------------------------------------------------
# Response dataclass
# ---------------------------------------------------------------------------

class TestResponse:
    def test_construction_text_only(self) -> None:
        blocks = [TextBlock(text="hello")]
        resp = Response(
            content=blocks,
            stop_reason="end_turn",
            input_tokens=10,
            output_tokens=5,
        )
        assert resp.stop_reason == "end_turn"
        assert resp.input_tokens == 10
        assert resp.output_tokens == 5
        assert len(resp.content) == 1

    def test_construction_tool_use(self) -> None:
        blocks = [ToolUseBlock(id="tu_1", name="read", input={"path": "/f"})]
        resp = Response(
            content=blocks,
            stop_reason="tool_use",
            input_tokens=20,
            output_tokens=15,
        )
        assert resp.stop_reason == "tool_use"

    def test_tool_use_blocks_property(self) -> None:
        blocks = [
            TextBlock(text="Let me read that."),
            ToolUseBlock(id="tu_1", name="read", input={"path": "/f"}),
            ToolUseBlock(id="tu_2", name="grep", input={"pattern": "TODO"}),
        ]
        resp = Response(
            content=blocks,
            stop_reason="tool_use",
            input_tokens=30,
            output_tokens=20,
        )
        tu_blocks = resp.tool_use_blocks
        assert len(tu_blocks) == 2
        assert all(isinstance(b, ToolUseBlock) for b in tu_blocks)

    def test_text_content_property(self) -> None:
        blocks = [TextBlock(text="first"), TextBlock(text="second")]
        resp = Response(
            content=blocks,
            stop_reason="end_turn",
            input_tokens=5,
            output_tokens=3,
        )
        assert resp.text_content == "first\nsecond"

    def test_text_content_empty(self) -> None:
        blocks = [ToolUseBlock(id="tu_1", name="bash", input={})]
        resp = Response(
            content=blocks,
            stop_reason="tool_use",
            input_tokens=5,
            output_tokens=3,
        )
        assert resp.text_content == ""


# ---------------------------------------------------------------------------
# AnthropicProvider construction
# ---------------------------------------------------------------------------

class TestAnthropicProviderConstruction:
    def test_default_model(self) -> None:
        with patch("neoagent.providers.anthropic.AsyncAnthropic"):
            provider = AnthropicProvider(api_key="sk-test")
            assert provider.model == "claude-sonnet-4-20250514"

    def test_custom_model(self) -> None:
        with patch("neoagent.providers.anthropic.AsyncAnthropic"):
            provider = AnthropicProvider(api_key="sk-test", model="claude-opus-4-20250514")
            assert provider.model == "claude-opus-4-20250514"

    def test_is_provider_subclass(self) -> None:
        with patch("neoagent.providers.anthropic.AsyncAnthropic"):
            provider = AnthropicProvider(api_key="sk-test")
            assert isinstance(provider, Provider)

    def test_get_context_window_sonnet(self) -> None:
        with patch("neoagent.providers.anthropic.AsyncAnthropic"):
            provider = AnthropicProvider(api_key="sk-test", model="claude-sonnet-4-20250514")
            assert provider.get_context_window() == 200_000

    def test_get_context_window_unknown_model(self) -> None:
        with patch("neoagent.providers.anthropic.AsyncAnthropic"):
            provider = AnthropicProvider(api_key="sk-test", model="claude-future-99")
            # Falls back to conservative default
            assert provider.get_context_window() == 200_000


# ---------------------------------------------------------------------------
# AnthropicProvider.create() — response conversion
# ---------------------------------------------------------------------------

def _make_fake_anthropic_response(
    stop_reason: str = "end_turn",
    blocks: list | None = None,
    input_tokens: int = 10,
    output_tokens: int = 5,
) -> MagicMock:
    """Build a fake anthropic SDK response object."""
    if blocks is None:
        text_block = MagicMock()
        text_block.type = "text"
        text_block.text = "I'm done."
        blocks = [text_block]

    usage = MagicMock()
    usage.input_tokens = input_tokens
    usage.output_tokens = output_tokens

    resp = MagicMock()
    resp.stop_reason = stop_reason
    resp.content = blocks
    resp.usage = usage
    return resp


class TestAnthropicProviderCreate:
    @pytest.fixture
    def mock_client(self) -> MagicMock:
        client = MagicMock()
        client.messages = MagicMock()
        client.messages.create = AsyncMock()
        return client

    @pytest.fixture
    def provider(self, mock_client: MagicMock) -> AnthropicProvider:
        with patch("neoagent.providers.anthropic.AsyncAnthropic", return_value=mock_client):
            p = AnthropicProvider(api_key="sk-test")
        p._client = mock_client
        return p

    async def test_create_text_response(self, provider: AnthropicProvider, mock_client: MagicMock) -> None:
        mock_client.messages.create.return_value = _make_fake_anthropic_response()

        messages = [Message(role="user", content="hello")]
        response = await provider.create(
            system="You are helpful.",
            messages=messages,
            tools=[],
        )

        assert isinstance(response, Response)
        assert response.stop_reason == "end_turn"
        assert len(response.content) == 1
        assert isinstance(response.content[0], TextBlock)
        assert response.content[0].text == "I'm done."
        assert response.input_tokens == 10
        assert response.output_tokens == 5

    async def test_create_tool_use_response(self, provider: AnthropicProvider, mock_client: MagicMock) -> None:
        tool_block = MagicMock()
        tool_block.type = "tool_use"
        tool_block.id = "tu_abc"
        tool_block.name = "read"
        tool_block.input = {"path": "/tmp/file.txt"}

        mock_client.messages.create.return_value = _make_fake_anthropic_response(
            stop_reason="tool_use",
            blocks=[tool_block],
            input_tokens=25,
            output_tokens=12,
        )

        messages = [Message(role="user", content="read /tmp/file.txt")]
        response = await provider.create(
            system="You are helpful.",
            messages=messages,
            tools=[{"name": "read", "description": "Reads a file", "input_schema": {}}],
        )

        assert response.stop_reason == "tool_use"
        assert len(response.tool_use_blocks) == 1
        block = response.tool_use_blocks[0]
        assert isinstance(block, ToolUseBlock)
        assert block.id == "tu_abc"
        assert block.name == "read"
        assert block.input == {"path": "/tmp/file.txt"}

    async def test_create_passes_system_prompt(self, provider: AnthropicProvider, mock_client: MagicMock) -> None:
        mock_client.messages.create.return_value = _make_fake_anthropic_response()

        await provider.create(
            system="Custom system prompt.",
            messages=[Message(role="user", content="hi")],
            tools=[],
        )

        call_kwargs = mock_client.messages.create.call_args.kwargs
        assert call_kwargs["system"] == "Custom system prompt."

    async def test_create_passes_tools(self, provider: AnthropicProvider, mock_client: MagicMock) -> None:
        mock_client.messages.create.return_value = _make_fake_anthropic_response()

        tools = [{"name": "bash", "description": "Run bash", "input_schema": {}}]
        await provider.create(
            system="sys",
            messages=[Message(role="user", content="go")],
            tools=tools,
        )

        call_kwargs = mock_client.messages.create.call_args.kwargs
        assert call_kwargs["tools"] == tools

    async def test_create_message_serialization(self, provider: AnthropicProvider, mock_client: MagicMock) -> None:
        """String-content messages serialize to {"role": ..., "content": ...}."""
        mock_client.messages.create.return_value = _make_fake_anthropic_response()

        messages = [Message(role="user", content="hello")]
        await provider.create(system="sys", messages=messages, tools=[])

        call_kwargs = mock_client.messages.create.call_args.kwargs
        serialized = call_kwargs["messages"]
        assert serialized == [{"role": "user", "content": "hello"}]

    async def test_create_unknown_block_type_skipped(
        self, provider: AnthropicProvider, mock_client: MagicMock
    ) -> None:
        """Unrecognized block types from the API are silently skipped."""
        unknown_block = MagicMock()
        unknown_block.type = "image"  # not supported in v1

        text_block = MagicMock()
        text_block.type = "text"
        text_block.text = "here is an image (ignored)"

        mock_client.messages.create.return_value = _make_fake_anthropic_response(
            blocks=[unknown_block, text_block]
        )

        response = await provider.create(
            system="sys",
            messages=[Message(role="user", content="show me an image")],
            tools=[],
        )

        # Only the text block should be in the result
        assert len(response.content) == 1
        assert isinstance(response.content[0], TextBlock)
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/providers/test_anthropic_provider.py -v`
Expected: FAIL with `ImportError: cannot import name 'Provider' from 'neoagent.providers.base'`

- [ ] **Step 3: Write implementation**

`neoagent/neoagent/providers/base.py`
```python
from __future__ import annotations

from abc import ABC, abstractmethod
from dataclasses import dataclass, field

from neoagent.core.types import ContentBlock, TextBlock, ToolUseBlock


@dataclass
class Response:
    """Normalized response from a model provider."""

    content: list[ContentBlock]
    stop_reason: str  # "end_turn" | "tool_use" | "max_tokens"
    input_tokens: int
    output_tokens: int

    @property
    def tool_use_blocks(self) -> list[ToolUseBlock]:
        return [b for b in self.content if isinstance(b, ToolUseBlock)]

    @property
    def text_content(self) -> str:
        parts = [b.text for b in self.content if isinstance(b, TextBlock)]
        return "\n".join(parts)


class Provider(ABC):
    """Abstract base class for model API providers."""

    @abstractmethod
    async def create(
        self,
        system: str,
        messages: list,
        tools: list,
    ) -> Response:
        """Call the model and return a normalized Response."""
        ...

    @abstractmethod
    def get_context_window(self) -> int:
        """Return the model's context window size in tokens."""
        ...
```

`neoagent/neoagent/providers/anthropic.py`
```python
from __future__ import annotations

from anthropic import AsyncAnthropic

from neoagent.core.types import ContentBlock, Message, TextBlock, ToolUseBlock
from neoagent.providers.base import Provider, Response

# Context window sizes by model family prefix
_CONTEXT_WINDOWS: dict[str, int] = {
    "claude-opus-4":    200_000,
    "claude-sonnet-4":  200_000,
    "claude-haiku-4":   200_000,
    "claude-3-5-sonnet": 200_000,
    "claude-3-5-haiku":  200_000,
    "claude-3-opus":    200_000,
}

_DEFAULT_CONTEXT_WINDOW = 200_000


def _get_context_window(model: str) -> int:
    for prefix, size in _CONTEXT_WINDOWS.items():
        if model.startswith(prefix):
            return size
    return _DEFAULT_CONTEXT_WINDOW


def _serialize_messages(messages: list[Message]) -> list[dict]:
    """Convert internal Message objects to the Anthropic API wire format."""
    serialized = []
    for msg in messages:
        if isinstance(msg.content, str):
            serialized.append({"role": msg.role, "content": msg.content})
        else:
            # list[ContentBlock] — serialize each block
            blocks = []
            for block in msg.content:
                if isinstance(block, TextBlock):
                    blocks.append({"type": "text", "text": block.text})
                elif isinstance(block, ToolUseBlock):
                    blocks.append({
                        "type": "tool_use",
                        "id": block.id,
                        "name": block.name,
                        "input": block.input,
                    })
                else:
                    # ToolResultBlock
                    blocks.append({
                        "type": "tool_result",
                        "tool_use_id": block.tool_use_id,
                        "content": block.content,
                        "is_error": block.is_error,
                    })
            serialized.append({"role": msg.role, "content": blocks})
    return serialized


def _parse_content_blocks(raw_blocks: list) -> list[ContentBlock]:
    """Convert Anthropic SDK content blocks to internal ContentBlock types."""
    parsed: list[ContentBlock] = []
    for block in raw_blocks:
        block_type = getattr(block, "type", None)
        if block_type == "text":
            parsed.append(TextBlock(text=block.text))
        elif block_type == "tool_use":
            parsed.append(ToolUseBlock(
                id=block.id,
                name=block.name,
                input=block.input,
            ))
        # Unrecognized block types (e.g. "image") are silently skipped
    return parsed


class AnthropicProvider(Provider):
    """Thin wrapper around the Anthropic AsyncAnthropic client."""

    def __init__(
        self,
        api_key: str,
        model: str = "claude-sonnet-4-20250514",
        max_tokens: int = 8096,
    ) -> None:
        self._client = AsyncAnthropic(api_key=api_key)
        self.model = model
        self.max_tokens = max_tokens

    def get_context_window(self) -> int:
        return _get_context_window(self.model)

    async def create(
        self,
        system: str,
        messages: list[Message],
        tools: list[dict],
    ) -> Response:
        serialized_messages = _serialize_messages(messages)

        raw = await self._client.messages.create(
            model=self.model,
            max_tokens=self.max_tokens,
            system=system,
            messages=serialized_messages,
            tools=tools,
        )

        content = _parse_content_blocks(raw.content)

        return Response(
            content=content,
            stop_reason=raw.stop_reason,
            input_tokens=raw.usage.input_tokens,
            output_tokens=raw.usage.output_tokens,
        )
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/providers/test_anthropic_provider.py -v`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add neoagent/neoagent/providers/base.py \
        neoagent/neoagent/providers/anthropic.py \
        neoagent/tests/providers/__init__.py \
        neoagent/tests/providers/test_anthropic_provider.py
git commit -m "feat: add Provider ABC and AnthropicProvider with response conversion"
```
