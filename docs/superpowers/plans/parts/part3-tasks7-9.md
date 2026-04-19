# neoagent v1 实现计划 — Part 3: Prompt System + Context Compression + Query Loop

> **For agentic workers:** Use `superpowers:subagent-driven-development` to execute this plan task-by-task.
> Each Task is independently executable. Steps use checkbox (`- [ ]`) syntax for tracking.

**范围**: Task 7 (PromptSystem) + Task 8 (ContextCompressor) + Task 9 (QueryLoop)

**前置依赖**:
- `neoagent/core/types.py` — `Message`, `Turn`, `ToolCall`, `ToolResult`, `ConversationResult` (Task 2)
- `neoagent/providers/base.py` — `Provider`, `Response` (Task 3)
- `neoagent/tools/registry.py` — `ToolRegistry` (Task 5)

**目录结构**（本 Part 涉及）:
```
neoagent/
└── core/
    ├── prompt.py      # Task 7
    ├── compress.py    # Task 8
    └── loop.py        # Task 9
tests/
└── core/
    ├── test_prompt.py
    ├── test_compress.py
    └── test_loop.py
```

---

## Task 7: Prompt System (`neoagent/core/prompt.py`)

### Files

- `neoagent/neoagent/core/prompt.py`
- `neoagent/tests/core/test_prompt.py`

---

### Step 1 — 写失败测试

文件路径: `neoagent/tests/core/test_prompt.py`

```python
from __future__ import annotations

import pytest

from neoagent.core.prompt import PromptBuilder, PromptSection


# ---------------------------------------------------------------------------
# PromptSection 构造
# ---------------------------------------------------------------------------

class TestPromptSection:
    def test_static_string_section(self) -> None:
        sec = PromptSection(name="identity", content="你是 neoagent。", priority=0)
        assert sec.name == "identity"
        assert sec.content == "你是 neoagent。"
        assert sec.priority == 0
        assert sec.is_static is True

    def test_dynamic_callable_section(self) -> None:
        sec = PromptSection(
            name="env",
            content=lambda: "cwd=/tmp",
            priority=10,
            is_static=False,
        )
        assert sec.is_static is False
        assert callable(sec.content)

    def test_default_is_static_true(self) -> None:
        sec = PromptSection(name="rules", content="rule 1", priority=1)
        assert sec.is_static is True


# ---------------------------------------------------------------------------
# PromptBuilder.add_section / remove_section
# ---------------------------------------------------------------------------

class TestPromptBuilderMutation:
    def test_add_section(self) -> None:
        builder = PromptBuilder()
        sec = PromptSection(name="identity", content="hi", priority=0)
        builder.add_section(sec)
        assert len(builder._sections) == 1

    def test_add_multiple_sections(self) -> None:
        builder = PromptBuilder()
        builder.add_section(PromptSection(name="a", content="A", priority=0))
        builder.add_section(PromptSection(name="b", content="B", priority=1))
        assert len(builder._sections) == 2

    def test_remove_existing_section(self) -> None:
        builder = PromptBuilder()
        builder.add_section(PromptSection(name="identity", content="hi", priority=0))
        builder.add_section(PromptSection(name="rules", content="rules", priority=1))
        builder.remove_section("identity")
        assert len(builder._sections) == 1
        assert builder._sections[0].name == "rules"

    def test_remove_nonexistent_section_is_noop(self) -> None:
        builder = PromptBuilder()
        builder.add_section(PromptSection(name="identity", content="hi", priority=0))
        builder.remove_section("nonexistent")  # must not raise
        assert len(builder._sections) == 1

    def test_add_duplicate_name_raises(self) -> None:
        builder = PromptBuilder()
        builder.add_section(PromptSection(name="identity", content="v1", priority=0))
        with pytest.raises(ValueError, match="identity"):
            builder.add_section(PromptSection(name="identity", content="v2", priority=0))


# ---------------------------------------------------------------------------
# PromptBuilder.build() — 排序规则
# ---------------------------------------------------------------------------

class TestPromptBuilderBuildOrdering:
    def test_static_before_dynamic(self) -> None:
        """静态 section 必须在动态 section 之前，与 priority 无关。"""
        builder = PromptBuilder()
        # 动态但 priority=0（比静态 priority=5 更小）
        builder.add_section(
            PromptSection(name="env", content=lambda: "dynamic", priority=0, is_static=False)
        )
        builder.add_section(
            PromptSection(name="rules", content="static", priority=5, is_static=True)
        )
        result = builder.build()
        rules_pos = result.index("# rules")
        env_pos = result.index("# env")
        assert rules_pos < env_pos, "静态 section 应出现在动态 section 之前"

    def test_static_sections_sorted_by_priority(self) -> None:
        builder = PromptBuilder()
        builder.add_section(PromptSection(name="b_sec", content="B", priority=2, is_static=True))
        builder.add_section(PromptSection(name="a_sec", content="A", priority=1, is_static=True))
        builder.add_section(PromptSection(name="c_sec", content="C", priority=3, is_static=True))
        result = builder.build()
        a_pos = result.index("# a_sec")
        b_pos = result.index("# b_sec")
        c_pos = result.index("# c_sec")
        assert a_pos < b_pos < c_pos

    def test_dynamic_sections_sorted_by_priority(self) -> None:
        builder = PromptBuilder()
        builder.add_section(
            PromptSection(name="d2", content=lambda: "D2", priority=20, is_static=False)
        )
        builder.add_section(
            PromptSection(name="d1", content=lambda: "D1", priority=11, is_static=False)
        )
        result = builder.build()
        d1_pos = result.index("# d1")
        d2_pos = result.index("# d2")
        assert d1_pos < d2_pos

    def test_empty_builder_returns_empty_string(self) -> None:
        builder = PromptBuilder()
        assert builder.build() == ""


# ---------------------------------------------------------------------------
# PromptBuilder.build() — 输出格式
# ---------------------------------------------------------------------------

class TestPromptBuilderBuildFormat:
    def test_single_section_format(self) -> None:
        builder = PromptBuilder()
        builder.add_section(PromptSection(name="identity", content="你是助手。", priority=0))
        result = builder.build()
        assert result == "# identity\n你是助手。"

    def test_two_sections_separated_by_blank_line(self) -> None:
        builder = PromptBuilder()
        builder.add_section(PromptSection(name="a", content="AAA", priority=0))
        builder.add_section(PromptSection(name="b", content="BBB", priority=1))
        result = builder.build()
        assert result == "# a\nAAA\n\n# b\nBBB"

    def test_callable_content_is_invoked(self) -> None:
        call_count = 0

        def dynamic() -> str:
            nonlocal call_count
            call_count += 1
            return f"call #{call_count}"

        builder = PromptBuilder()
        builder.add_section(
            PromptSection(name="dyn", content=dynamic, priority=10, is_static=False)
        )
        result = builder.build()
        assert "# dyn\ncall #1" in result
        assert call_count == 1

    def test_callable_reinvoked_each_build(self) -> None:
        """每次 build() 都会重新调用 Callable，不缓存结果。"""
        counter = {"n": 0}

        def ticker() -> str:
            counter["n"] += 1
            return str(counter["n"])

        builder = PromptBuilder()
        builder.add_section(
            PromptSection(name="tick", content=ticker, priority=10, is_static=False)
        )
        first = builder.build()
        second = builder.build()
        assert first != second
        assert counter["n"] == 2

    def test_mixed_static_and_dynamic_format(self) -> None:
        builder = PromptBuilder()
        builder.add_section(PromptSection(name="identity", content="I am agent.", priority=0))
        builder.add_section(
            PromptSection(name="env", content=lambda: "cwd=/home", priority=10, is_static=False)
        )
        result = builder.build()
        assert result == "# identity\nI am agent.\n\n# env\ncwd=/home"
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/core/test_prompt.py -v`
Expected: FAIL with `ImportError: cannot import name 'PromptBuilder' from 'neoagent.core.prompt'`

- [ ] **Step 3: Write implementation**

`neoagent/neoagent/core/prompt.py`
```python
from __future__ import annotations

from dataclasses import dataclass, field
from typing import Callable


@dataclass
class PromptSection:
    """A single named block of content in the system prompt.

    Static sections (is_static=True) contain pre-computed string content and
    are placed before dynamic sections to maximise prompt-caching efficiency.
    Dynamic sections contain a Callable that is invoked fresh on every build().
    Within each group, sections are ordered by ascending priority (lower = earlier).
    """

    name: str
    content: str | Callable[[], str]
    priority: int
    is_static: bool = True


class PromptBuilder:
    """Assembles a system prompt from ordered PromptSection instances.

    Ordering rules (applied inside build()):
    1. Static sections (is_static=True) come before dynamic sections.
    2. Within each group, lower priority value appears first.

    Output format:
        # {section.name}
        {content}

    Sections are separated by a single blank line (\\n\\n).
    """

    def __init__(self) -> None:
        self._sections: list[PromptSection] = []

    # ------------------------------------------------------------------
    # Mutation helpers
    # ------------------------------------------------------------------

    def add_section(self, section: PromptSection) -> None:
        """Add a section.  Raises ValueError if a section with the same name
        already exists (names must be unique to allow remove_section() by name).
        """
        existing_names = {s.name for s in self._sections}
        if section.name in existing_names:
            raise ValueError(
                f"A section named '{section.name}' already exists. "
                "Remove the existing one before adding a replacement."
            )
        self._sections.append(section)

    def remove_section(self, name: str) -> None:
        """Remove the section with the given name.  No-op if not found."""
        self._sections = [s for s in self._sections if s.name != name]

    # ------------------------------------------------------------------
    # Build
    # ------------------------------------------------------------------

    def build(self) -> str:
        """Return the assembled system prompt string.

        Static sections are sorted by priority and emitted first; dynamic
        sections follow, also sorted by priority.  Callable content is invoked
        at build time (not cached).
        """
        if not self._sections:
            return ""

        sorted_sections = sorted(
            self._sections,
            # Primary key: is_static=True → 0 (first), False → 1 (last)
            # Secondary key: priority ascending
            key=lambda s: (0 if s.is_static else 1, s.priority),
        )

        parts: list[str] = []
        for sec in sorted_sections:
            content = sec.content if isinstance(sec.content, str) else sec.content()
            parts.append(f"# {sec.name}\n{content}")

        return "\n\n".join(parts)
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/core/test_prompt.py -v`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add neoagent/neoagent/core/prompt.py \
        neoagent/tests/core/test_prompt.py
git commit -m "feat: add PromptBuilder with static/dynamic section ordering"
```

---

## Task 8: Context Compression (`neoagent/core/compress.py`)

### Files

- `neoagent/neoagent/core/compress.py`
- `neoagent/tests/core/test_compress.py`

---

### Step 1 — 写失败测试

文件路径: `neoagent/tests/core/test_compress.py`

```python
from __future__ import annotations

import json
from unittest.mock import AsyncMock, MagicMock, patch

import pytest

from neoagent.core.types import Message, TextBlock, ToolResultBlock, ToolUseBlock
from neoagent.core.compress import ContextCompressor


# ---------------------------------------------------------------------------
# Helpers
# ---------------------------------------------------------------------------

def _user(text: str) -> Message:
    return Message(role="user", content=text)


def _assistant(text: str) -> Message:
    return Message(role="assistant", content=text)


def _assistant_with_tool(tool_id: str, tool_name: str) -> Message:
    return Message(
        role="assistant",
        content=[ToolUseBlock(id=tool_id, name=tool_name, input={})],
    )


def _tool_result(tool_id: str, output: str = "ok") -> Message:
    return Message(
        role="user",
        content=[ToolResultBlock(tool_use_id=tool_id, content=output)],
    )


def _make_provider(summary: str = "SUMMARY") -> MagicMock:
    """Return a mock Provider whose create() returns a text response."""
    response = MagicMock()
    response.text_content = summary
    provider = MagicMock()
    provider.create = AsyncMock(return_value=response)
    return provider


# ---------------------------------------------------------------------------
# estimate_tokens
# ---------------------------------------------------------------------------

class TestEstimateTokens:
    def test_empty_messages(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        assert compressor.estimate_tokens([]) == 0

    def test_string_content_nonzero(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        msgs = [_user("Hello, how are you?")]
        tokens = compressor.estimate_tokens(msgs)
        assert tokens > 0

    def test_longer_message_more_tokens(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        short = [_user("Hi")]
        long = [_user("Hi " * 200)]
        assert compressor.estimate_tokens(long) > compressor.estimate_tokens(short)

    def test_list_content_serialized(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        msgs = [
            Message(
                role="assistant",
                content=[TextBlock(text="Let me check."), ToolUseBlock(id="tu_1", name="read", input={"path": "/f"})],
            )
        ]
        tokens = compressor.estimate_tokens(msgs)
        assert tokens > 0


# ---------------------------------------------------------------------------
# estimate_tools_tokens
# ---------------------------------------------------------------------------

class TestEstimateToolsTokens:
    def test_empty_schemas(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        assert compressor.estimate_tools_tokens([]) == 0

    def test_nonempty_schemas_nonzero(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        schemas = [
            {"name": "read", "description": "Read a file", "input_schema": {"type": "object", "properties": {}}}
        ]
        tokens = compressor.estimate_tools_tokens(schemas)
        assert tokens > 0

    def test_more_schemas_more_tokens(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        one_schema = [{"name": "read", "description": "x", "input_schema": {}}]
        many_schemas = one_schema * 20
        assert compressor.estimate_tools_tokens(many_schemas) > compressor.estimate_tools_tokens(one_schema)


# ---------------------------------------------------------------------------
# should_compress
# ---------------------------------------------------------------------------

class TestShouldCompress:
    def test_below_threshold_returns_false(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        # 1 short message, budget=100_000 → well below 70%
        msgs = [_user("Hi")]
        assert compressor.should_compress(msgs, [], context_budget=100_000) is False

    def test_above_threshold_returns_true(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        # Manufacture a message that definitely exceeds 70% of a tiny budget
        long_text = "word " * 500
        msgs = [_user(long_text)]
        # Budget of 50 tokens → 35 tokens threshold; 500 words >> 35 tokens
        assert compressor.should_compress(msgs, [], context_budget=50) is True

    def test_tools_tokens_counted_toward_budget(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        # 10 big tool schemas should push total over a tiny budget
        big_schemas = [
            {"name": f"tool_{i}", "description": "x" * 200, "input_schema": {}}
            for i in range(10)
        ]
        msgs = [_user("short")]
        assert compressor.should_compress(msgs, big_schemas, context_budget=50) is True

    def test_exactly_at_threshold_not_compressed(self) -> None:
        """Budget exactly at 70% boundary: should NOT compress (> not >=)."""
        compressor = ContextCompressor(provider=_make_provider())
        # Use a very large budget so 1 message is well below 70%
        msgs = [_user("a")]
        assert compressor.should_compress(msgs, [], context_budget=1_000_000) is False


# ---------------------------------------------------------------------------
# _sanitize_tool_pairs
# ---------------------------------------------------------------------------

class TestSanitizeToolPairs:
    def test_clean_conversation_unchanged(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        msgs = [
            _user("go"),
            _assistant_with_tool("tu_1", "read"),
            _tool_result("tu_1"),
            _assistant("done"),
        ]
        result = compressor._sanitize_tool_pairs(msgs)
        assert len(result) == 4

    def test_orphan_tool_use_removed(self) -> None:
        """tool_use block with no matching tool_result is removed."""
        compressor = ContextCompressor(provider=_make_provider())
        msgs = [
            _user("go"),
            _assistant_with_tool("tu_orphan", "read"),  # no result follows
            _assistant("done"),
        ]
        result = compressor._sanitize_tool_pairs(msgs)
        # The assistant message containing the orphan tool_use should be stripped/cleaned
        for msg in result:
            if isinstance(msg.content, list):
                for block in msg.content:
                    assert not isinstance(block, ToolUseBlock), \
                        "Orphan ToolUseBlock should have been removed"

    def test_orphan_tool_result_removed(self) -> None:
        """tool_result block referencing unknown call_id is removed."""
        compressor = ContextCompressor(provider=_make_provider())
        msgs = [
            _user("go"),
            _tool_result("tu_missing"),  # no corresponding tool_use
            _assistant("done"),
        ]
        result = compressor._sanitize_tool_pairs(msgs)
        for msg in result:
            if isinstance(msg.content, list):
                for block in msg.content:
                    assert not isinstance(block, ToolResultBlock), \
                        "Orphan ToolResultBlock should have been removed"

    def test_matched_pairs_preserved(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        msgs = [
            _user("go"),
            _assistant_with_tool("tu_1", "bash"),
            _tool_result("tu_1", "output"),
        ]
        result = compressor._sanitize_tool_pairs(msgs)
        # Both tool_use and tool_result should survive
        has_use = any(
            isinstance(msg.content, list) and any(isinstance(b, ToolUseBlock) for b in msg.content)
            for msg in result
        )
        has_result = any(
            isinstance(msg.content, list) and any(isinstance(b, ToolResultBlock) for b in msg.content)
            for msg in result
        )
        assert has_use and has_result

    def test_empty_messages(self) -> None:
        compressor = ContextCompressor(provider=_make_provider())
        assert compressor._sanitize_tool_pairs([]) == []


# ---------------------------------------------------------------------------
# compress — happy path
# ---------------------------------------------------------------------------

class TestCompress:
    async def test_compress_returns_message_list(self) -> None:
        provider = _make_provider(summary="Summarized earlier context.")
        compressor = ContextCompressor(provider=provider)
        msgs = [_user("start")] + [_user(f"msg {i}") for i in range(10)]
        result = await compressor.compress(msgs, context_budget=200_000)
        assert isinstance(result, list)
        assert len(result) > 0

    async def test_first_message_preserved(self) -> None:
        provider = _make_provider(summary="SUMMARY")
        compressor = ContextCompressor(provider=provider)
        first = _user("This is the very first message that must not be lost.")
        msgs = [first] + [_user(f"msg {i}") for i in range(8)]
        result = await compressor.compress(msgs, context_budget=200_000)
        # First message should always be at index 0
        assert result[0].content == first.content

    async def test_summary_injected_as_user_message(self) -> None:
        provider = _make_provider(summary="CONTEXT_SUMMARY")
        compressor = ContextCompressor(provider=provider)
        msgs = [_user("initial")] + [_user(f"turn {i}") for i in range(6)]
        result = await compressor.compress(msgs, context_budget=200_000)
        # The summary should appear somewhere in the compressed messages
        all_text = " ".join(
            msg.content if isinstance(msg.content, str) else
            " ".join(b.text for b in msg.content if isinstance(b, TextBlock))
            for msg in result
        )
        assert "CONTEXT_SUMMARY" in all_text

    async def test_success_resets_consecutive_failures(self) -> None:
        provider = _make_provider(summary="ok")
        compressor = ContextCompressor(provider=provider)
        compressor._consecutive_failures = 2  # seed some prior failures
        msgs = [_user("a")] + [_user(f"b{i}") for i in range(4)]
        await compressor.compress(msgs, context_budget=200_000)
        assert compressor._consecutive_failures == 0


# ---------------------------------------------------------------------------
# compress — circuit breaker (熔断降级)
# ---------------------------------------------------------------------------

class TestCompressCircuitBreaker:
    async def test_failure_increments_counter(self) -> None:
        provider = MagicMock()
        provider.create = AsyncMock(side_effect=RuntimeError("API down"))
        compressor = ContextCompressor(provider=provider, max_failures=3)
        msgs = [_user("a")] + [_user(f"b{i}") for i in range(4)]
        await compressor.compress(msgs, context_budget=200_000)
        assert compressor._consecutive_failures == 1

    async def test_circuit_breaks_after_max_failures(self) -> None:
        """After max_failures consecutive failures, compress() falls back to
        truncation instead of calling the provider again."""
        provider = MagicMock()
        provider.create = AsyncMock(side_effect=RuntimeError("API down"))
        compressor = ContextCompressor(provider=provider, max_failures=3)
        compressor._consecutive_failures = 3  # already at max

        msgs = [_user("first")] + [_user(f"extra {i}") for i in range(10)]
        result = await compressor.compress(msgs, context_budget=200_000)

        # Provider should NOT have been called (circuit open)
        provider.create.assert_not_called()
        # Result should still be a valid (truncated) list
        assert isinstance(result, list)
        assert len(result) > 0

    async def test_fallback_preserves_first_message(self) -> None:
        provider = MagicMock()
        provider.create = AsyncMock(side_effect=RuntimeError("dead"))
        compressor = ContextCompressor(provider=provider, max_failures=1)
        compressor._consecutive_failures = 1

        first = _user("anchor message")
        msgs = [first] + [_user(f"extra {i}") for i in range(20)]
        result = await compressor.compress(msgs, context_budget=200_000)

        assert result[0].content == first.content
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/core/test_compress.py -v`
Expected: FAIL with `ImportError: cannot import name 'ContextCompressor' from 'neoagent.core.compress'`

- [ ] **Step 3: Write implementation**

`neoagent/neoagent/core/compress.py`
```python
from __future__ import annotations

import json
import logging
from typing import TYPE_CHECKING

import tiktoken

from neoagent.core.types import Message, TextBlock, ToolResultBlock, ToolUseBlock

if TYPE_CHECKING:
    from neoagent.providers.base import Provider

logger = logging.getLogger(__name__)

# Number of recent messages to always retain verbatim (tail of conversation).
_KEEP_RECENT = 6

# Tiktoken encoding for token estimation (cl100k_base — used by Claude models).
_ENCODING = tiktoken.get_encoding("cl100k_base")

# Prompt sent to the LLM when requesting a summary.
_SUMMARIZE_SYSTEM = (
    "You are a context compression assistant. "
    "Summarize the following conversation history concisely, "
    "preserving all key facts, decisions, tool outputs, and context "
    "that would be needed to continue the conversation seamlessly. "
    "Output only the summary — no preamble, no meta-commentary."
)


def _message_to_text(msg: Message) -> str:
    """Render a Message to a plain-text string for token counting."""
    if isinstance(msg.content, str):
        return f"{msg.role}: {msg.content}"
    parts: list[str] = [f"{msg.role}:"]
    for block in msg.content:
        if isinstance(block, TextBlock):
            parts.append(block.text)
        elif isinstance(block, ToolUseBlock):
            parts.append(f"[tool_use id={block.id} name={block.name} input={json.dumps(block.input)}]")
        elif isinstance(block, ToolResultBlock):
            parts.append(f"[tool_result id={block.tool_use_id} content={block.content}]")
    return " ".join(parts)


class ContextCompressor:
    """LLM-based context compression with circuit-breaker fallback.

    Strategy:
    - Keep the first message (system anchor / initial user task) verbatim.
    - Keep the last _KEEP_RECENT messages verbatim (recent context).
    - Summarise the middle section via the provider.
    - On consecutive provider failures >= max_failures, fall back to simple
      oldest-message truncation (drop messages from the middle until under budget).

    Hermes-derived _sanitize_tool_pairs() is run after every compression to
    ensure the output is a legally-structured Anthropic conversation.
    """

    def __init__(self, provider: Provider, max_failures: int = 3) -> None:
        self._provider = provider
        self._max_failures = max_failures
        self._consecutive_failures: int = 0

    # ------------------------------------------------------------------
    # Token estimation
    # ------------------------------------------------------------------

    def estimate_tokens(self, messages: list[Message]) -> int:
        """Estimate the token count of a list of messages using tiktoken."""
        total = 0
        for msg in messages:
            text = _message_to_text(msg)
            total += len(_ENCODING.encode(text))
        return total

    def estimate_tools_tokens(self, schemas: list[dict]) -> int:
        """Estimate the token overhead of tool schemas passed to the model."""
        if not schemas:
            return 0
        serialized = json.dumps(schemas)
        return len(_ENCODING.encode(serialized))

    # ------------------------------------------------------------------
    # Trigger check
    # ------------------------------------------------------------------

    def should_compress(
        self,
        messages: list[Message],
        schemas: list[dict],
        context_budget: int,
    ) -> bool:
        """Return True when the combined token count exceeds 70% of the budget."""
        msg_tokens = self.estimate_tokens(messages)
        tool_tokens = self.estimate_tools_tokens(schemas)
        return (msg_tokens + tool_tokens) > context_budget * 0.7

    # ------------------------------------------------------------------
    # Public compress entry-point
    # ------------------------------------------------------------------

    async def compress(
        self,
        messages: list[Message],
        context_budget: int,
    ) -> list[Message]:
        """Compress the conversation history and return the reduced list.

        If the circuit-breaker is open (consecutive_failures >= max_failures),
        falls back to truncation without calling the provider.
        """
        if len(messages) <= 1:
            return messages

        # Circuit-breaker check — fall back to truncation if open.
        if self._consecutive_failures >= self._max_failures:
            logger.warning(
                "ContextCompressor: circuit breaker open after %d consecutive failures; "
                "falling back to truncation.",
                self._consecutive_failures,
            )
            return self._truncate_oldest(messages)

        try:
            compressed = await self._llm_compress(messages)
            self._consecutive_failures = 0
            return compressed
        except Exception as exc:
            self._consecutive_failures += 1
            logger.warning(
                "ContextCompressor: LLM compress failed (%s). "
                "Consecutive failures: %d / %d. Falling back to truncation.",
                exc,
                self._consecutive_failures,
                self._max_failures,
            )
            return self._truncate_oldest(messages)

    # ------------------------------------------------------------------
    # LLM-based compression (happy path)
    # ------------------------------------------------------------------

    async def _llm_compress(self, messages: list[Message]) -> list[Message]:
        """Summarise the middle section of the conversation via the provider."""
        if len(messages) <= _KEEP_RECENT + 1:
            # Not enough messages to compress meaningfully.
            return messages

        anchor = messages[0]
        middle = messages[1 : len(messages) - _KEEP_RECENT]
        recent = messages[len(messages) - _KEEP_RECENT :]

        # Build a plain-text representation of the middle section.
        middle_text = "\n".join(_message_to_text(m) for m in middle)
        summarize_prompt = [Message(role="user", content=middle_text)]

        response = await self._provider.create(
            system=_SUMMARIZE_SYSTEM,
            messages=summarize_prompt,
            tools=[],
        )
        summary = response.text_content.strip() or "(no summary)"

        summary_message = Message(
            role="user",
            content=f"[Context summary from earlier in the conversation]\n{summary}",
        )

        compressed = [anchor, summary_message, *recent]
        return self._sanitize_tool_pairs(compressed)

    # ------------------------------------------------------------------
    # Truncation fallback (circuit-breaker open)
    # ------------------------------------------------------------------

    def _truncate_oldest(self, messages: list[Message]) -> list[Message]:
        """Drop messages from the middle, preserving anchor and recent tail."""
        if len(messages) <= _KEEP_RECENT + 1:
            return messages

        anchor = messages[0]
        recent = messages[len(messages) - _KEEP_RECENT :]
        # Simply discard everything between anchor and recent tail.
        result = [anchor, *recent]
        return self._sanitize_tool_pairs(result)

    # ------------------------------------------------------------------
    # Hermes-derived tool-pair sanitiser
    # ------------------------------------------------------------------

    def _sanitize_tool_pairs(self, messages: list[Message]) -> list[Message]:
        """Remove orphaned tool_use and tool_result blocks.

        Pass 1 — collect all tool_use ids that have a corresponding tool_result.
        Pass 2 — collect all tool_result call_ids that have a corresponding tool_use.
        Pass 3 — rewrite messages, dropping orphan blocks.
        """
        # Collect all tool_use ids and tool_result call_ids present.
        all_tool_use_ids: set[str] = set()
        all_result_ids: set[str] = set()

        for msg in messages:
            if isinstance(msg.content, list):
                for block in msg.content:
                    if isinstance(block, ToolUseBlock):
                        all_tool_use_ids.add(block.id)
                    elif isinstance(block, ToolResultBlock):
                        all_result_ids.add(block.tool_use_id)

        # A tool_use is "matched" iff there is a tool_result for it.
        matched_use_ids = all_tool_use_ids & all_result_ids
        # A tool_result is "matched" iff its call_id has a corresponding tool_use.
        matched_result_ids = all_result_ids & all_tool_use_ids

        sanitized: list[Message] = []
        for msg in messages:
            if isinstance(msg.content, str):
                sanitized.append(msg)
                continue

            clean_blocks = []
            for block in msg.content:
                if isinstance(block, ToolUseBlock) and block.id not in matched_use_ids:
                    # Orphan tool_use — discard.
                    logger.debug(
                        "_sanitize_tool_pairs: dropping orphan tool_use id=%s name=%s",
                        block.id,
                        block.name,
                    )
                    continue
                if isinstance(block, ToolResultBlock) and block.tool_use_id not in matched_result_ids:
                    # Orphan tool_result — discard.
                    logger.debug(
                        "_sanitize_tool_pairs: dropping orphan tool_result id=%s",
                        block.tool_use_id,
                    )
                    continue
                clean_blocks.append(block)

            # Only include the message if it still has content after stripping orphans.
            if clean_blocks:
                sanitized.append(Message(role=msg.role, content=clean_blocks))
            elif isinstance(msg.content, list) and not clean_blocks:
                # All blocks were orphans; drop the entire message.
                pass

        return sanitized
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/core/test_compress.py -v`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add neoagent/neoagent/core/compress.py \
        neoagent/tests/core/test_compress.py
git commit -m "feat: add ContextCompressor with LLM summarisation and circuit-breaker fallback"
```

---

## Task 9: Query Loop (`neoagent/core/loop.py`)

### Files

- `neoagent/neoagent/core/loop.py`
- `neoagent/tests/core/test_loop.py`

---

### Step 1 — 写失败测试

文件路径: `neoagent/tests/core/test_loop.py`

```python
from __future__ import annotations

from typing import Callable
from unittest.mock import AsyncMock, MagicMock, call, patch

import pytest

from neoagent.core.loop import QueryLoop
from neoagent.core.types import (
    ConversationResult,
    Message,
    TextBlock,
    ToolCall,
    ToolResult,
    ToolUseBlock,
    Turn,
)


# ---------------------------------------------------------------------------
# Helpers — mock factories
# ---------------------------------------------------------------------------

def _text_response(text: str = "done") -> MagicMock:
    """Simulate a provider Response with stop_reason='end_turn'."""
    resp = MagicMock()
    resp.stop_reason = "end_turn"
    resp.content = [MagicMock(spec=TextBlock, type="text", text=text)]
    resp.text_content = text
    resp.tool_use_blocks = []
    return resp


def _tool_use_response(tool_id: str, tool_name: str, tool_input: dict) -> MagicMock:
    """Simulate a provider Response with stop_reason='tool_use'."""
    tu_block = MagicMock(spec=ToolUseBlock)
    tu_block.type = "tool_use"
    tu_block.id = tool_id
    tu_block.name = tool_name
    tu_block.input = tool_input

    resp = MagicMock()
    resp.stop_reason = "tool_use"
    resp.content = [tu_block]
    resp.text_content = ""
    resp.tool_use_blocks = [tu_block]
    return resp


def _max_tokens_response() -> MagicMock:
    resp = MagicMock()
    resp.stop_reason = "max_tokens"
    resp.content = []
    resp.text_content = ""
    resp.tool_use_blocks = []
    return resp


def _make_provider(*responses) -> MagicMock:
    """Provider whose create() yields responses in sequence."""
    provider = MagicMock()
    provider.get_context_window.return_value = 200_000
    provider.create = AsyncMock(side_effect=list(responses))
    return provider


def _make_registry(tool_results: list[ToolResult] | None = None) -> MagicMock:
    """ToolRegistry that returns preset results and exposes get_schemas()."""
    results = tool_results or []
    registry = MagicMock()
    registry.get_schemas.return_value = []
    registry.execute = AsyncMock(return_value=results)
    return registry


def _make_prompt_builder(system_text: str = "You are neoagent.") -> MagicMock:
    builder = MagicMock()
    builder.build.return_value = system_text
    return builder


# ---------------------------------------------------------------------------
# QueryLoop construction
# ---------------------------------------------------------------------------

class TestQueryLoopConstruction:
    def test_default_context_budget_from_provider(self) -> None:
        provider = _make_provider()
        loop = QueryLoop(
            provider=provider,
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
        )
        assert loop.context_budget == 200_000

    def test_explicit_context_budget_overrides_provider(self) -> None:
        provider = _make_provider()
        loop = QueryLoop(
            provider=provider,
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
            context_budget=50_000,
        )
        assert loop.context_budget == 50_000

    def test_default_max_turns(self) -> None:
        loop = QueryLoop(
            provider=_make_provider(),
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
        )
        assert loop.max_turns == 30

    def test_custom_max_turns(self) -> None:
        loop = QueryLoop(
            provider=_make_provider(),
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
            max_turns=5,
        )
        assert loop.max_turns == 5


# ---------------------------------------------------------------------------
# QueryLoop.run() — normal completion
# ---------------------------------------------------------------------------

class TestQueryLoopNormalCompletion:
    async def test_single_turn_end_turn(self) -> None:
        """Model returns end_turn on the first call → completed with 1 turn."""
        provider = _make_provider(_text_response("All done."))
        loop = QueryLoop(
            provider=provider,
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
        )
        messages = [Message(role="user", content="Hello")]
        result = await loop.run(messages)

        assert isinstance(result, ConversationResult)
        assert result.reason == "completed"
        assert len(result.turns) == 1
        assert result.turns[0].stop_reason == "end_turn"

    async def test_provider_called_with_system_and_messages(self) -> None:
        provider = _make_provider(_text_response())
        builder = _make_prompt_builder("SYSTEM PROMPT")
        loop = QueryLoop(
            provider=provider,
            tool_registry=_make_registry(),
            prompt_builder=builder,
        )
        messages = [Message(role="user", content="go")]
        await loop.run(messages)

        provider.create.assert_called_once()
        call_kwargs = provider.create.call_args.kwargs
        assert call_kwargs["system"] == "SYSTEM PROMPT"

    async def test_tool_schemas_passed_to_provider(self) -> None:
        provider = _make_provider(_text_response())
        schemas = [{"name": "read", "description": "Read file", "input_schema": {}}]
        registry = _make_registry()
        registry.get_schemas.return_value = schemas

        loop = QueryLoop(
            provider=provider,
            tool_registry=registry,
            prompt_builder=_make_prompt_builder(),
        )
        await loop.run([Message(role="user", content="go")])

        call_kwargs = provider.create.call_args.kwargs
        assert call_kwargs["tools"] == schemas


# ---------------------------------------------------------------------------
# QueryLoop.run() — max_turns
# ---------------------------------------------------------------------------

class TestQueryLoopMaxTurns:
    async def test_max_turns_returns_max_turns_reason(self) -> None:
        """If model keeps returning tool_use indefinitely, loop exits at max_turns."""
        # Provide more tool_use responses than max_turns; registry always returns ok
        tool_responses = [
            _tool_use_response(f"tu_{i}", "bash", {"cmd": "ls"}) for i in range(10)
        ]
        provider = _make_provider(*tool_responses)
        registry = _make_registry([ToolResult(call_id="tu_0", output="ok")])

        loop = QueryLoop(
            provider=provider,
            tool_registry=registry,
            prompt_builder=_make_prompt_builder(),
            max_turns=3,
        )
        result = await loop.run([Message(role="user", content="go")])

        assert result.reason == "max_turns"
        assert len(result.turns) == 3

    async def test_provider_called_max_turns_times(self) -> None:
        tool_responses = [
            _tool_use_response(f"tu_{i}", "bash", {"cmd": "x"}) for i in range(5)
        ]
        provider = _make_provider(*tool_responses)
        registry = _make_registry([ToolResult(call_id="x", output="out")])

        loop = QueryLoop(
            provider=provider,
            tool_registry=registry,
            prompt_builder=_make_prompt_builder(),
            max_turns=3,
        )
        await loop.run([Message(role="user", content="go")])
        assert provider.create.call_count == 3


# ---------------------------------------------------------------------------
# QueryLoop.run() — tool call flow
# ---------------------------------------------------------------------------

class TestQueryLoopToolCallFlow:
    async def test_tool_executed_and_result_fed_back(self) -> None:
        """Model calls a tool, result is fed back, model then returns end_turn."""
        tool_resp = _tool_use_response("tu_1", "read", {"path": "/f"})
        end_resp = _text_response("file read done")

        provider = _make_provider(tool_resp, end_resp)
        tool_result = ToolResult(call_id="tu_1", output="file contents")
        registry = _make_registry([tool_result])

        loop = QueryLoop(
            provider=provider,
            tool_registry=registry,
            prompt_builder=_make_prompt_builder(),
        )
        result = await loop.run([Message(role="user", content="read /f")])

        assert result.reason == "completed"
        assert len(result.turns) == 2
        # First turn: tool_use
        assert result.turns[0].stop_reason == "tool_use"
        assert len(result.turns[0].tool_calls) == 1
        assert result.turns[0].tool_calls[0].id == "tu_1"
        # Second turn: end_turn
        assert result.turns[1].stop_reason == "end_turn"

    async def test_registry_execute_called_with_tool_calls(self) -> None:
        tool_resp = _tool_use_response("tu_abc", "grep", {"pattern": "TODO"})
        end_resp = _text_response("searched")
        provider = _make_provider(tool_resp, end_resp)
        registry = _make_registry([ToolResult(call_id="tu_abc", output="found 3")])

        loop = QueryLoop(
            provider=provider,
            tool_registry=registry,
            prompt_builder=_make_prompt_builder(),
        )
        await loop.run([Message(role="user", content="search")])

        registry.execute.assert_called_once()
        calls_arg = registry.execute.call_args.args[0]
        assert len(calls_arg) == 1
        assert calls_arg[0].id == "tu_abc"
        assert calls_arg[0].name == "grep"

    async def test_tool_results_appended_to_messages(self) -> None:
        """After tool execution, a user message with tool_result blocks is added."""
        tool_resp = _tool_use_response("tu_1", "bash", {"cmd": "ls"})
        end_resp = _text_response("listed")
        provider = _make_provider(tool_resp, end_resp)
        registry = _make_registry([ToolResult(call_id="tu_1", output="file.txt")])

        loop = QueryLoop(
            provider=provider,
            tool_registry=registry,
            prompt_builder=_make_prompt_builder(),
        )
        await loop.run([Message(role="user", content="list files")])

        # Second call to provider should include the tool result in messages
        second_call_kwargs = provider.create.call_args_list[1].kwargs
        second_messages = second_call_kwargs["messages"]
        # The last message before the second call should contain the tool_result
        last_msg = second_messages[-1]
        assert last_msg.role == "user"
        assert isinstance(last_msg.content, list)
        from neoagent.core.types import ToolResultBlock
        assert any(isinstance(b, ToolResultBlock) for b in last_msg.content)


# ---------------------------------------------------------------------------
# QueryLoop.run() — on_turn callback
# ---------------------------------------------------------------------------

class TestQueryLoopOnTurnCallback:
    async def test_on_turn_called_once_per_turn(self) -> None:
        tool_resp = _tool_use_response("tu_1", "read", {"path": "/f"})
        end_resp = _text_response("done")
        provider = _make_provider(tool_resp, end_resp)
        registry = _make_registry([ToolResult(call_id="tu_1", output="content")])

        on_turn_calls: list[Turn] = []

        def on_turn(turn: Turn) -> None:
            on_turn_calls.append(turn)

        loop = QueryLoop(
            provider=provider,
            tool_registry=registry,
            prompt_builder=_make_prompt_builder(),
            on_turn=on_turn,
        )
        result = await loop.run([Message(role="user", content="go")])

        assert len(on_turn_calls) == 2  # once per turn
        assert on_turn_calls[0].stop_reason == "tool_use"
        assert on_turn_calls[1].stop_reason == "end_turn"

    async def test_on_turn_receives_turn_object(self) -> None:
        provider = _make_provider(_text_response("hello"))
        received: list[Turn] = []

        loop = QueryLoop(
            provider=provider,
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
            on_turn=received.append,
        )
        await loop.run([Message(role="user", content="hi")])

        assert len(received) == 1
        assert isinstance(received[0], Turn)

    async def test_no_on_turn_callback_is_fine(self) -> None:
        """Loop runs normally when on_turn is None."""
        provider = _make_provider(_text_response())
        loop = QueryLoop(
            provider=provider,
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
            on_turn=None,
        )
        result = await loop.run([Message(role="user", content="hi")])
        assert result.reason == "completed"


# ---------------------------------------------------------------------------
# QueryLoop.run() — max_tokens retry
# ---------------------------------------------------------------------------

class TestQueryLoopMaxTokensRetry:
    async def test_max_tokens_triggers_retry(self) -> None:
        """On max_tokens, loop retries with a lower max_tokens parameter."""
        max_tokens_resp = _max_tokens_response()
        retry_resp = _text_response("done after retry")
        provider = _make_provider(max_tokens_resp, retry_resp)

        loop = QueryLoop(
            provider=provider,
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
        )
        result = await loop.run([Message(role="user", content="go")])

        # Both calls happened
        assert provider.create.call_count == 2
        assert result.reason == "completed"

    async def test_max_tokens_retry_uses_lower_max_tokens(self) -> None:
        """The retry call should pass a smaller max_tokens than the first call."""
        max_tokens_resp = _max_tokens_response()
        retry_resp = _text_response("ok")
        provider = _make_provider(max_tokens_resp, retry_resp)

        loop = QueryLoop(
            provider=provider,
            tool_registry=_make_registry(),
            prompt_builder=_make_prompt_builder(),
        )
        await loop.run([Message(role="user", content="go")])

        first_call_kwargs = provider.create.call_args_list[0].kwargs
        second_call_kwargs = provider.create.call_args_list[1].kwargs
        first_max = first_call_kwargs.get("max_tokens", None)
        second_max = second_call_kwargs.get("max_tokens", None)
        # If max_tokens was explicitly set on both calls, the retry should be lower.
        if first_max is not None and second_max is not None:
            assert second_max < first_max
```

- [ ] **Step 2: Run test to verify failure**

Run: `cd neoagent && python -m pytest tests/core/test_loop.py -v`
Expected: FAIL with `ImportError: cannot import name 'QueryLoop' from 'neoagent.core.loop'`

- [ ] **Step 3: Write implementation**

`neoagent/neoagent/core/loop.py`
```python
from __future__ import annotations

import logging
from typing import Callable

from neoagent.core.compress import ContextCompressor
from neoagent.core.prompt import PromptBuilder
from neoagent.core.types import (
    ConversationResult,
    Message,
    ToolCall,
    ToolResult,
    ToolResultBlock,
    ToolUseBlock,
    Turn,
)
from neoagent.providers.base import Provider, Response
from neoagent.tools.registry import ToolRegistry

logger = logging.getLogger(__name__)

# When a max_tokens response is received, retry with this fraction of the
# original max_tokens value.  Mirrors Claude Code's behaviour.
_MAX_TOKENS_RETRY_FRACTION = 0.5
_DEFAULT_MAX_TOKENS = 8096


class QueryLoop:
    """The main agentic loop.

    Drives the model in a simple while-loop until one of:
    - The model returns stop_reason='end_turn'  → reason='completed'
    - max_turns is exhausted                    → reason='max_turns'

    On every turn:
    1. Check context budget; compress if needed (DeerFlow trigger pattern).
    2. Build system prompt via PromptBuilder.
    3. Call provider.create().
    4. On 'max_tokens': retry once with lower max_tokens (Claude Code pattern).
    5. On 'tool_use': execute tools via ToolRegistry; append results to messages.
    6. Invoke on_turn callback if provided.
    """

    def __init__(
        self,
        provider: Provider,
        tool_registry: ToolRegistry,
        prompt_builder: PromptBuilder,
        max_turns: int = 30,
        context_budget: int = 0,
        on_turn: Callable[[Turn], None] | None = None,
    ) -> None:
        self._provider = provider
        self._registry = tool_registry
        self._prompt_builder = prompt_builder
        self.max_turns = max_turns
        # 0 means "ask the provider".
        self.context_budget = context_budget if context_budget > 0 else provider.get_context_window()
        self._on_turn = on_turn
        self._compressor = ContextCompressor(provider=provider)

    # ------------------------------------------------------------------
    # Public entry-point
    # ------------------------------------------------------------------

    async def run(self, messages: list[Message]) -> ConversationResult:
        """Execute the agent loop and return a ConversationResult."""
        # Work on a copy so the caller's list is not mutated.
        msgs: list[Message] = list(messages)
        turns: list[Turn] = []

        for _turn_idx in range(self.max_turns):
            # --- 1. Context compression check ---
            schemas = self._registry.get_schemas()
            if self._compressor.should_compress(msgs, schemas, self.context_budget):
                logger.debug("QueryLoop: compressing context at turn %d", _turn_idx)
                msgs = await self._compressor.compress(msgs, self.context_budget)

            # --- 2. Build system prompt ---
            system = self._prompt_builder.build()

            # --- 3. Call the model ---
            response = await self._provider.create(
                system=system,
                messages=msgs,
                tools=schemas,
                max_tokens=_DEFAULT_MAX_TOKENS,
            )

            # --- 4. Handle max_tokens: retry once with a lower cap ---
            if response.stop_reason == "max_tokens":
                response = await self._retry_with_lower_max(system, msgs, schemas)

            # --- 5. Build the Turn record ---
            tool_calls = self._extract_tool_calls(response)
            tool_results: list[ToolResult] = []

            if response.stop_reason == "end_turn" or not tool_calls:
                # Model is done — record this turn and return.
                turn = Turn(
                    response=Message(role="assistant", content=response.content),
                    tool_calls=[],
                    tool_results=[],
                    stop_reason="end_turn",
                )
                turns.append(turn)
                if self._on_turn:
                    self._on_turn(turn)
                return ConversationResult(turns=turns, reason="completed")

            # --- 6. Execute tools ---
            tool_results = await self._registry.execute(tool_calls)

            # --- 7. Append assistant + user (tool_result) messages ---
            assistant_msg = Message(role="assistant", content=response.content)
            msgs.append(assistant_msg)
            tool_result_msg = self._build_tool_result_message(tool_results)
            msgs.append(tool_result_msg)

            # --- 8. Record turn ---
            turn = Turn(
                response=assistant_msg,
                tool_calls=tool_calls,
                tool_results=tool_results,
                stop_reason="tool_use",
            )
            turns.append(turn)
            if self._on_turn:
                self._on_turn(turn)

        return ConversationResult(turns=turns, reason="max_turns")

    # ------------------------------------------------------------------
    # Internal helpers
    # ------------------------------------------------------------------

    async def _retry_with_lower_max(
        self,
        system: str,
        messages: list[Message],
        tools: list[dict],
    ) -> Response:
        """Retry the model call with a reduced max_tokens (Claude Code pattern)."""
        lower_max = max(1, int(_DEFAULT_MAX_TOKENS * _MAX_TOKENS_RETRY_FRACTION))
        logger.warning(
            "QueryLoop: max_tokens hit — retrying with max_tokens=%d", lower_max
        )
        return await self._provider.create(
            system=system,
            messages=messages,
            tools=tools,
            max_tokens=lower_max,
        )

    def _extract_tool_calls(self, response: Response) -> list[ToolCall]:
        """Convert ToolUseBlock objects in the response to ToolCall records."""
        return [
            ToolCall(id=block.id, name=block.name, input=block.input)
            for block in response.tool_use_blocks
        ]

    def _build_tool_result_message(self, results: list[ToolResult]) -> Message:
        """Wrap a list of ToolResults into a user Message with ToolResultBlocks."""
        blocks = [
            ToolResultBlock(
                tool_use_id=r.call_id,
                content=r.output,
                is_error=r.is_error,
            )
            for r in results
        ]
        return Message(role="user", content=blocks)
```

- [ ] **Step 4: Run test to verify pass**

Run: `cd neoagent && python -m pytest tests/core/test_loop.py -v`
Expected: PASS — all tests green

- [ ] **Step 5: Commit**

```bash
git add neoagent/neoagent/core/loop.py \
        neoagent/tests/core/test_loop.py
git commit -m "feat: add QueryLoop with tool execution, compression trigger, and max_tokens retry"
```

---

## Part 3 完成检查

运行全部 Part 3 测试：

```bash
cd neoagent && python -m pytest tests/core/test_prompt.py tests/core/test_compress.py tests/core/test_loop.py -v
```

Expected: all green.  Then run the full suite to ensure no regressions:

```bash
cd neoagent && python -m pytest -v
```
