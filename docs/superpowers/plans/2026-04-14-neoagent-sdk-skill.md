# neoagent SDK Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create ~/.claude/skills/neoagent/SKILL.md — a Claude Code skill that guides LLMs to build production-ready agents with the neoagent SDK.

**Architecture:** Single markdown skill file containing: module architecture diagram, 10-scenario reference table, 10 production-ready code skeletons, anti-patterns, and quality checklist. No code to compile or test — validation is: skill loads in Claude Code and triggers on neoagent-related prompts.

**Tech Stack:** Markdown (Claude Code SKILL.md format), neoagent SDK v0.1.0 (local install)

---

## Task 1: Scaffold + frontmatter + overview

**Goal:** Create `~/.claude/skills/neoagent/SKILL.md` with correct frontmatter, trigger description, install section, and quick-start overview.

- [ ] Create directory `~/.claude/skills/neoagent/` if it does not already exist.

- [ ] Create the file `~/.claude/skills/neoagent/SKILL.md` with the following exact content as the starting point (subsequent tasks will append sections):

```
---
name: neoagent
description: >
  Use when building agents with the neoagent SDK, understanding how SDK modules
  connect, adding capabilities (memory / custom tools / multi-agent / hooks / HTTP
  channel), or reading/debugging existing neoagent code.
---

# neoagent SDK

> Python async agent framework. Production-ready. Local install only (not on PyPI).

## Install

```bash
pip install -e /path/to/neoagent
```

Replace `/path/to/neoagent` with the absolute path to the neoagent repository on disk.

## Quick Start

```python
import asyncio
from neoagent import NeoAgent, NeoAgentConfig

config = NeoAgentConfig(
    api_key="sk-ant-...",          # or os.environ["ANTHROPIC_API_KEY"]
    model="claude-sonnet-4-20250514",
    system_prompt="You are a helpful assistant.",
    max_turns=30,
    context_budget=80_000,         # tokens — set explicitly, never rely on default 0
)
agent = NeoAgent(config)

async def main():
    reply = await agent.chat("Hello, what can you do?")
    print(reply)

asyncio.run(main())
```

**Entry points:**
- `agent.chat(message)` → single-turn, returns `str`
- `agent.run(messages, max_turns=N)` → full conversation, returns `ConversationResult`

---
```

- [ ] Verify the file exists at `~/.claude/skills/neoagent/SKILL.md` and the frontmatter `description` matches the trigger string exactly.

---

## Task 2: Module architecture diagram

**Goal:** Append the full 12-module architecture diagram and wiki→neoagent mapping table to the SKILL.md.

- [ ] Append the following section to `~/.claude/skills/neoagent/SKILL.md`:

````
## Module Architecture

All 12 modules and their data-flow relationships:

```
NeoAgentConfig (config.py)
    │
    ▼
NeoAgent (agent.py)  ◄── sole entry point: chat() / run()
    │
    ├── [prompt-system] PromptBuilder (core/prompt.py)
    │       add_section(PromptSection)       permanent prompt blocks
    │       register_skill(name, section)    register skill (inactive)
    │       activate_skill(name)             inject skill → system prompt
    │       deactivate_skill(name)           remove skill from system prompt
    │
    ├── [query-loop] QueryLoop (core/loop.py)   ← execution engine / state machine
    │       │
    │       ├── [tool-system] Provider (providers/)    API calls (Anthropic / OpenAI)
    │       │
    │       ├── ToolExecutor
    │       │       ToolRegistry                  active tools (LLM-visible)
    │       │       DeferredToolRegistry          tool pool (hidden until promoted)
    │       │           ▲ tool_search built-in promotes on demand
    │       │
    │       ├── [hooks] HookManager (hooks.py)    pre/post interception
    │       │       hook(type, handler)           register via method
    │       │       @agent.on(type)               register via decorator
    │       │
    │       └── [context-management] ContextCompressor (core/compress.py)  auto compress
    │
    ├── [memory-system] MemoryManager (memory/)    cross-session memory
    │       MemoryExtractor         extract memories after turns
    │       MemoryRetriever         retrieve relevant memories
    │       MemoryStore             persist to disk
    │       (enabled when memory_dir + memory_project_key are set in config)
    │
    ├── [runtime-state / session-recovery] Session + SessionState (session.py)
    │       Session.create()                new session
    │       agent.resume(session_id)        load from storage (preferred API)
    │       Session.resume(id, storage)     low-level classmethod
    │       JsonFileStorage                 persist to session_dir
    │       (enabled when session_dir is set in config)
    │
    ├── [hooks] EventBus (events.py)         publish/subscribe observability bus
    │       agent._event_bus.subscribe(EventClass, handler)
    │       agent._event_bus.unsubscribe(EventClass, handler)
    │       agent._event_bus.emit(event)
    │       agent.event_bus                 public property alias
    │
    └── [evaluation-observability] observe.py + observe_subscriber.py
            Observer(console=True, log_dir=Path(...))
            ObserverSubscriber(observer).attach(agent._event_bus)

─── Independent extension modules ────────────────────────────────────────

[mcp-skills] MCP (mcp/)                    [multi-agent] Multi-Agent (multi/)
    MCPClient                                  Orchestrator(config, max_depth=2)
    MCPTool                                    register_worker(WorkerCard(...))
    MCPTransport (stdio only)                  await orchestrator.run(message)
    await agent.add_mcp_server(name, cmd)      await orchestrator.close()
    → tools auto-injected to DeferredRegistry

[channel-remote] Channel (channels/)       [evaluation-observability] Eval (eval/)
    FastAPIChannel(agent, host, port)          EvalRunner(agent)
    await channel.serve_forever()             await runner.run(cases)  → EvalReport
    POST /v1/run  /v1/run/stream               EvalCase(messages=[...], assertion=fn)
```
````

- [ ] Verify the diagram section was appended and renders without broken markdown (no unclosed code fences).

---

## Task 3: wiki→neoagent mapping table

**Goal:** Append the 12-row wiki-to-source mapping table to SKILL.md.

- [ ] Append the following section to `~/.claude/skills/neoagent/SKILL.md`:

```
## wiki Module → neoagent Source

| wiki module | neoagent implementation |
|-------------|------------------------|
| query-loop | `neoagent/core/loop.py` |
| prompt-system | `neoagent/core/prompt.py` (PromptBuilder + PromptSection) |
| tool-system | `neoagent/tools/` — BaseTool, ToolRegistry, DeferredToolRegistry, ToolExecutor |
| context-management | `neoagent/core/compress.py` — ContextCompressor |
| memory-system | `neoagent/memory/` — MemoryExtractor, MemoryRetriever, MemoryStore |
| runtime-state | `neoagent/session.py` — SessionState, Session |
| session-recovery | `neoagent/session.py` — Session.resume() + JsonFileStorage |
| hooks | `neoagent/hooks.py` — HookManager, HookResult; `neoagent/events.py` — EventBus |
| mcp-skills | `neoagent/mcp/` — MCPClient, MCPTool, MCPTransport |
| multi-agent | `neoagent/multi/` — Orchestrator, WorkerCard, WorkerRegistry |
| channel-remote | `neoagent/channels/` — FastAPIChannel, BaseChannel |
| evaluation-observability | `neoagent/eval/` — EvalRunner, EvalCase, EvalReport; `neoagent/observe.py` — Observer; `neoagent/observe_subscriber.py` — ObserverSubscriber |
```

- [ ] Verify table renders (12 data rows, 2 header rows, consistent pipe alignment is not required but pipes must be present).

---

## Task 4: Scenario quick reference table

**Goal:** Append the 10-scenario reference table to SKILL.md.

- [ ] Append the following section to `~/.claude/skills/neoagent/SKILL.md`:

```
## Scenario Quick Reference

| # | Scenario | Core modules | Key API / config fields |
|---|----------|-------------|------------------------|
| 1 | Minimal agent | NeoAgent + NeoAgentConfig | `api_key`, `model`, `system_prompt`, `max_turns`, `context_budget` |
| 2 | Custom tool | BaseTool + register_tool() | Subclass `BaseTool`; implement `async execute()`; set `permission`; call `agent.register_tool(tool)` |
| 3 | Cross-session memory | MemoryManager | `memory_dir=Path(...)`, `memory_project_key="..."` in config |
| 4 | Dynamic skill (prompt injection) | PromptBuilder | `agent._prompt_builder.register_skill(name, PromptSection(...))` → `activate_skill(name)` |
| 5 | Event hook + interceptor | HookManager + EventBus | `agent.hook("pre_tool_call", handler)` or `@agent.on("pre_tool_call")`; `agent._event_bus.subscribe(EventClass, handler)` |
| 6 | MCP tools | mcp/ + DeferredToolRegistry | `await agent.add_mcp_server(name, command, env)` → tools auto-registered; LLM uses `tool_search` to promote |
| 7 | Multi-agent orchestration | Orchestrator + WorkerCard | `Orchestrator(config, max_depth=2)`; `register_worker(WorkerCard(...))`; `await orchestrator.run(msg)` |
| 8 | HTTP API server | FastAPIChannel | `FastAPIChannel(agent, host, port)` → `await channel.serve_forever()` |
| 9 | Session recovery (resume) | Session + JsonFileStorage | `session_dir=Path(...)` in config; `session = agent.resume(session_id)` → `await agent.chat(msg, session=session)` |
| 10 | Observability + eval | Observer + ObserverSubscriber + EvalRunner | `Observer(console=True, log_dir=Path(...))`; `ObserverSubscriber(obs).attach(agent._event_bus)`; `EvalRunner(agent).run(cases)` |
```

- [ ] Verify table has 10 data rows (scenarios 1–10) and all columns are populated.

---

## Task 5: Code skeletons — Scenarios 1–5

**Goal:** Append production-ready minimal code skeletons for scenarios 1–5 to SKILL.md.

- [ ] Append the following section to `~/.claude/skills/neoagent/SKILL.md`:

````
## Code Skeletons

> **Production-ready standard:** every skeleton below is wired correctly and safe to use as a starting point. Read the Anti-patterns section before adding your own code.

### Scenario 1 — Minimal agent

```python
import asyncio
import os
from neoagent import NeoAgent, NeoAgentConfig

config = NeoAgentConfig(
    api_key=os.environ["ANTHROPIC_API_KEY"],
    model="claude-sonnet-4-20250514",
    system_prompt="You are a helpful assistant.",
    max_turns=30,
    # Set context_budget explicitly — default 0 means no limit, which can cause
    # runaway context growth in long sessions.
    context_budget=80_000,
)
agent = NeoAgent(config)

async def main() -> None:
    reply: str = await agent.chat("Hello")
    print(reply)

asyncio.run(main())
```

### Scenario 2 — Custom tool

```python
import asyncio
import os
from pydantic import BaseModel
from neoagent import NeoAgent, NeoAgentConfig
from neoagent.tools.base import BaseTool, ToolResult

class SearchInput(BaseModel):
    query: str
    max_results: int = 5

class SearchTool(BaseTool):
    name = "search"
    # description is a prompt for the LLM — be precise and unambiguous
    description = "Search a knowledge base and return up to max_results relevant passages."
    input_model = SearchInput
    # permission="ask" because this tool makes external network calls.
    # Only use "auto" for pure read operations with no side-effects and no
    # external calls (e.g., reading a local file you own).
    permission = "ask"
    # Only mark True if the tool has no shared mutable state — here it's
    # stateless, so concurrent calls are safe.
    is_concurrent_safe = True

    async def execute(self, input: SearchInput) -> ToolResult:
        # Replace with real search implementation
        results = [f"Result {i} for '{input.query}'" for i in range(input.max_results)]
        return ToolResult(call_id="", output="\n".join(results))

async def main() -> None:
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are a research assistant with access to a search tool.",
        max_turns=20,
        context_budget=60_000,
    )
    agent = NeoAgent(config)
    agent.register_tool(SearchTool())
    reply = await agent.chat("Search for Python async patterns")
    print(reply)

asyncio.run(main())
```

### Scenario 3 — Cross-session memory

```python
import asyncio
import os
from pathlib import Path
from neoagent import NeoAgent, NeoAgentConfig

# memory_dir + memory_project_key together enable MemoryManager.
# The agent extracts memories after each turn and retrieves relevant
# ones at the start of subsequent sessions — no manual calls needed.
config = NeoAgentConfig(
    api_key=os.environ["ANTHROPIC_API_KEY"],
    model="claude-sonnet-4-20250514",
    system_prompt="You are a personal assistant who remembers user preferences.",
    max_turns=30,
    context_budget=80_000,
    memory_dir=Path("~/.neoagent/memory").expanduser(),
    # memory_project_key namespaces memories so different projects don't bleed.
    memory_project_key="my-assistant",
)
agent = NeoAgent(config)

async def main() -> None:
    # First session — agent stores relevant facts
    await agent.chat("My preferred language is Python and I hate boilerplate.")
    # Second session (new NeoAgent instance with same config) — agent
    # automatically retrieves stored preferences and injects them.

asyncio.run(main())
```

### Scenario 4 — Dynamic skill (prompt injection)

```python
import asyncio
import os
from neoagent import NeoAgent, NeoAgentConfig
from neoagent.core.prompt import PromptSection

CODING_SKILL_CONTENT = """\
You are an expert Python developer. When writing code:
- Use type annotations on all function signatures.
- Prefer dataclasses / Pydantic over plain dicts.
- Write async code unless sync is explicitly required.
- Add docstrings on public functions.
"""

async def main() -> None:
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are a general assistant.",
        max_turns=20,
        context_budget=60_000,
    )
    agent = NeoAgent(config)

    # Register once — does NOT inject into prompt yet
    agent._prompt_builder.register_skill(
        "coding",
        PromptSection(name="coding", content=CODING_SKILL_CONTENT),
    )

    # Activate only when user switches to a coding task
    agent._prompt_builder.activate_skill("coding")
    reply = await agent.chat("Write a function that fetches JSON from a URL.")
    print(reply)

    # Deactivate when switching back to general mode
    agent._prompt_builder.deactivate_skill("coding")

asyncio.run(main())
```

### Scenario 5 — Event hook + interceptor

```python
import asyncio
import os
from neoagent import NeoAgent, NeoAgentConfig
from neoagent.hooks import HookResult, HookType
from neoagent.events import ToolCallEvent, TurnCompleteEvent

async def main() -> None:
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are an assistant with tool access.",
        max_turns=20,
        context_budget=60_000,
    )
    agent = NeoAgent(config)

    # ── Interceptor hook: runs BEFORE tool execution, can block the call ──
    # Use agent.hook() (method) or @agent.on() (decorator) — both are equivalent.
    @agent.on("pre_tool_call")
    async def guard_tool(event) -> HookResult:
        # Block calls to any tool starting with "write_" unless explicitly allowed
        if event.tool_name.startswith("write_"):
            return HookResult.block(reason="Write tools require explicit user approval")
        return HookResult.allow()

    # ── Observation event: fires AFTER tool execution, cannot block ──
    # Subscribe via the EventBus for passive observation (logging, metrics, etc.)
    def on_tool_call(event: ToolCallEvent) -> None:
        print(f"[tool] {event.tool_name} called with {event.input_data}")

    def on_turn_complete(event: TurnCompleteEvent) -> None:
        print(f"[turn] completed, usage={event.usage}")

    agent._event_bus.subscribe(ToolCallEvent, on_tool_call)
    agent._event_bus.subscribe(TurnCompleteEvent, on_turn_complete)

    reply = await agent.chat("Try to write something to disk")
    print(reply)

    # Clean up subscriptions before agent is discarded
    agent._event_bus.unsubscribe(ToolCallEvent, on_tool_call)
    agent._event_bus.unsubscribe(TurnCompleteEvent, on_turn_complete)

asyncio.run(main())
```
````

- [ ] Verify the section was appended. Count 5 code blocks (one per scenario). Verify no unclosed fences.

---

## Task 6: Code skeletons — Scenarios 6–10

**Goal:** Append production-ready minimal code skeletons for scenarios 6–10 to SKILL.md.

- [ ] Append the following section to `~/.claude/skills/neoagent/SKILL.md`:

````
### Scenario 6 — MCP tools

```python
import asyncio
import os
from neoagent import NeoAgent, NeoAgentConfig

async def main() -> None:
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are an assistant with access to filesystem tools.",
        max_turns=30,
        context_budget=80_000,
    )
    agent = NeoAgent(config)

    # add_mcp_server() connects once and keeps the connection alive for the
    # lifetime of the agent. The MCP client is owned by the agent — do NOT
    # call MCPClient directly or re-connect inside loops.
    #
    # Tools are injected into DeferredToolRegistry (hidden from LLM).
    # The built-in `tool_search` tool promotes matching tools on demand,
    # keeping the active tool list small.
    await agent.add_mcp_server(
        name="filesystem",                         # prefix for all tools: filesystem__read_file, etc.
        command=["npx", "@anthropic/mcp-server-filesystem", "/tmp"],
        env={"MCP_LOG_LEVEL": "error"},            # optional env vars for the subprocess
    )

    reply = await agent.chat("List files in /tmp and summarize what you find.")
    print(reply)

    # Clean up MCP connections when done
    await agent.remove_mcp_server("filesystem")

asyncio.run(main())
```

### Scenario 7 — Multi-agent orchestration

```python
import asyncio
import os
from neoagent import NeoAgentConfig
from neoagent.multi import Orchestrator, WorkerCard

async def main() -> None:
    # Orchestrator uses this config for the planning/routing agent.
    # Each worker agent gets its own NeoAgent instance with the same
    # api_key/model but its own system_prompt (from WorkerCard.instruction).
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are an orchestrator that delegates tasks to specialists.",
        max_turns=10,
        context_budget=60_000,
    )

    orchestrator = Orchestrator(
        config,
        max_depth=2,              # max recursion depth for sub-orchestration
        max_concurrent_workers=5, # semaphore limit on parallel worker agents
    )

    orchestrator.register_worker(WorkerCard(
        name="researcher",
        description="Searches for information and summarizes findings.",
        instruction="You are a research specialist. Find accurate, cited information.",
        tags=("research", "web", "summarize"),
        tools=("bash",),          # only tools in this tuple are available to this worker
    ))

    orchestrator.register_worker(WorkerCard(
        name="writer",
        description="Writes polished prose given research notes.",
        instruction="You are a technical writer. Write clearly and concisely.",
        tags=("writing", "editing"),
        tools=(),                 # read-only worker — no tools
    ))

    # run() dispatches to workers via SpawnWorkerTool; result is the final output
    result: str = await orchestrator.run(
        "Research Python async best practices and write a 3-paragraph summary."
    )
    print(result)

    # Always close — releases worker agent resources and MCP connections
    await orchestrator.close()

asyncio.run(main())
```

### Scenario 8 — HTTP API server

```python
import asyncio
import os
from neoagent import NeoAgent, NeoAgentConfig
from neoagent.channels import FastAPIChannel

async def main() -> None:
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are a helpful HTTP-accessible assistant.",
        max_turns=30,
        context_budget=80_000,
    )
    agent = NeoAgent(config)

    # FastAPIChannel wraps the agent in a FastAPI ASGI app.
    # Constructor accepts host/port — do NOT pass them again to serve_forever().
    channel = FastAPIChannel(
        agent,
        host="127.0.0.1",  # use 0.0.0.0 only if you intend external access
        port=8000,
        streaming=True,    # enable SSE streaming on POST /v1/run/stream
    )

    # serve_forever() calls start() internally, then blocks until stop() is called.
    # Endpoints:
    #   POST /v1/run         → {"message": "..."} → {"response": "..."}
    #   POST /v1/run/stream  → {"message": "..."} → SSE event stream
    await channel.serve_forever()

asyncio.run(main())
```

### Scenario 9 — Session recovery (resume)

```python
import asyncio
import os
from pathlib import Path
from neoagent import NeoAgent, NeoAgentConfig
from neoagent.session import Session

async def first_session() -> str:
    """Start a new session and return its ID for later resumption."""
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are a coding assistant.",
        max_turns=30,
        context_budget=80_000,
        # session_dir enables JsonFileStorage — sessions persist to disk.
        session_dir=Path("~/.neoagent/sessions").expanduser(),
    )
    agent = NeoAgent(config)
    session: Session = agent.new_session()
    await agent.chat("Start writing a Python web scraper.", session=session)
    # Caller stores session.id (e.g., in a database) for the next invocation
    return session.id

async def resume_session(session_id: str) -> None:
    """Resume a session from a previous run."""
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are a coding assistant.",
        max_turns=30,
        context_budget=80_000,
        session_dir=Path("~/.neoagent/sessions").expanduser(),
    )
    agent = NeoAgent(config)
    # agent.resume() uses the configured JsonFileStorage — raises KeyError if not found
    session: Session = agent.resume(session_id)
    await agent.chat("Continue — add error handling and retry logic.", session=session)

async def main() -> None:
    sid = await first_session()
    print(f"Session saved: {sid}")
    await resume_session(sid)

asyncio.run(main())
```

### Scenario 10 — Observability + eval

```python
import asyncio
import os
from pathlib import Path
from neoagent import NeoAgent, NeoAgentConfig
from neoagent.observe import Observer
from neoagent.observe_subscriber import ObserverSubscriber
from neoagent.eval.runner import EvalRunner, EvalCase
from neoagent.core.types import Message

async def main() -> None:
    config = NeoAgentConfig(
        api_key=os.environ["ANTHROPIC_API_KEY"],
        model="claude-sonnet-4-20250514",
        system_prompt="You are a helpful assistant.",
        max_turns=20,
        context_budget=60_000,
    )
    agent = NeoAgent(config)

    # ── Observer: attach before any chat() calls ──
    # Observer subscribes to all EventBus events and writes structured logs.
    # Always clean up with try/finally — otherwise log files stay open.
    observer = Observer(console=True, log_dir=Path("logs/"))
    subscriber = ObserverSubscriber(observer)
    subscriber.attach(agent._event_bus)

    try:
        # ── EvalRunner: quality measurement across test cases ──
        def assert_mentions_python(result: str) -> bool:
            return "python" in result.lower()

        cases = [
            EvalCase(
                messages=[Message(role="user", content="What is Python?")],
                assertion=assert_mentions_python,
            ),
            EvalCase(
                messages=[Message(role="user", content="Explain asyncio in one sentence.")],
                assertion=lambda r: "async" in r.lower() or "coroutine" in r.lower(),
            ),
        ]
        runner = EvalRunner(agent)
        report = await runner.run(cases)
        print(f"Passed: {report.passed}/{report.total} ({report.pass_rate:.0%})")
        for r in report.cases:
            status = "PASS" if r.passed else "FAIL"
            print(f"  [{status}] error={r.error}")
    finally:
        # Detach before closing — ensures no stale handlers on the EventBus
        subscriber.detach(agent._event_bus)
        observer.close()

asyncio.run(main())
```
````

- [ ] Verify 5 additional code blocks were appended (scenarios 6–10). Verify no unclosed fences.

---

## Task 7: Anti-patterns section

**Goal:** Append the 4 anti-patterns section (bad/good pairs) to SKILL.md.

- [ ] Append the following section to `~/.claude/skills/neoagent/SKILL.md`:

````
## Anti-patterns

These are the four most common mistakes LLMs make when generating neoagent code.
**Every generated code block should be checked against these before output.**

### 1 — Loop IO: creating connections inside a loop

```python
# BAD — creates a new MCPClient + subprocess for every item.
# This is a resource leak and a performance bottleneck.
for query in queries:
    await agent.add_mcp_server("search", ["python", "search_server.py"])
    result = await agent.chat(query)
    await agent.remove_mcp_server("search")

# GOOD — connect once, reuse for all calls.
# MCP connections are long-lived and expensive; bind them to agent lifetime.
await agent.add_mcp_server("search", ["python", "search_server.py"])
results = []
for query in queries:
    results.append(agent.chat(query))
# If the queries are independent, run them concurrently:
results = await asyncio.gather(*[agent.chat(q) for q in queries])
await agent.remove_mcp_server("search")
```

### 2 — Permission abuse: using `"auto"` on write operations

```python
# BAD — "auto" means the LLM can execute this tool without asking the user.
# Giving write/delete operations auto permission is a security risk.
class DeleteFileTool(BaseTool):
    name = "delete_file"
    description = "Deletes a file at the given path."
    permission = "auto"   # WRONG: destructive operation must require confirmation

# GOOD — write/delete/network-mutating operations always use "ask".
# Reserve "auto" only for pure reads with no external side effects.
class DeleteFileTool(BaseTool):
    name = "delete_file"
    description = "Deletes a file at the given path. Requires user confirmation."
    permission = "ask"    # user is prompted before execution

class ReadFileTool(BaseTool):
    name = "read_file"
    description = "Reads a local file and returns its content."
    permission = "auto"   # OK: read-only, no side effects
    is_concurrent_safe = True
```

### 3 — Resource leak: forgetting to detach Observer + close

```python
# BAD — observer's log file handle is never closed.
# EventBus holds a reference to subscriber's handlers — memory leak until GC.
observer = Observer(log_dir=Path("logs/"))
subscriber = ObserverSubscriber(observer)
subscriber.attach(agent._event_bus)
result = await agent.chat(message)
# program continues with leaked file handle + stale EventBus subscription

# GOOD — always use try/finally (or an async context manager) to guarantee cleanup.
observer = Observer(log_dir=Path("logs/"))
subscriber = ObserverSubscriber(observer)
subscriber.attach(agent._event_bus)
try:
    result = await agent.chat(message)
finally:
    subscriber.detach(agent._event_bus)  # removes handlers from EventBus
    observer.close()                     # flushes + closes log file
```

### 4 — Concurrent safety lie: `is_concurrent_safe=True` on stateful tools

```python
# BAD — the tool holds a shared connection object; concurrent calls will race.
class DatabaseTool(BaseTool):
    name = "db_query"
    description = "Queries the database."
    is_concurrent_safe = True  # WRONG: self._conn is shared mutable state

    def __init__(self):
        self._conn = create_db_connection()  # shared across concurrent calls

    async def execute(self, input):
        return ToolResult(call_id="", output=await self._conn.query(input.sql))

# GOOD — create a new connection per call, OR use a connection pool that is
# itself thread/coroutine safe, and then declare is_concurrent_safe=True.
class DatabaseTool(BaseTool):
    name = "db_query"
    description = "Queries the database."
    is_concurrent_safe = False  # conservative default for stateful tools

    async def execute(self, input):
        # Each call gets its own connection — safe for concurrent execution
        async with create_db_connection() as conn:
            result = await conn.query(input.sql)
        return ToolResult(call_id="", output=result)
```
````

- [ ] Verify 4 anti-pattern subsections (### 1 through ### 4) were appended, each with a BAD + GOOD code pair.

---

## Task 8: Production-ready checklist + deep reference table

**Goal:** Append the quality self-check checklist and the deep reference table to SKILL.md.

- [ ] Append the following section to `~/.claude/skills/neoagent/SKILL.md`:

```
## Production-Ready Checklist

Before submitting any neoagent code, verify every item below. If any item is
unchecked the code is **not** production-ready.

### Correctness
- [ ] `BaseTool.permission` is `"ask"` for tools with write, delete, or network-mutating operations. Only pure read-only, side-effect-free tools use `"auto"`.
- [ ] `is_concurrent_safe=True` is only set when the tool has no shared mutable state and no side effects that could conflict under concurrent execution.
- [ ] `context_budget` is set explicitly in `NeoAgentConfig` (not left at default `0`). For most tasks: `60_000`–`120_000` tokens.
- [ ] All `async def` methods use `await` consistently. No synchronous blocking IO (`open`, `requests.get`, etc.) inside `async def`.
- [ ] API keys come from environment variables (`os.environ["ANTHROPIC_API_KEY"]`), never hardcoded.

### Performance and resources
- [ ] MCP servers are added once per agent lifetime with `add_mcp_server()` — not inside loops.
- [ ] Independent tasks are run with `asyncio.gather()`, not sequential `await` in a loop.
- [ ] `Observer` + `ObserverSubscriber` are cleaned up with `try/finally`: `subscriber.detach(bus)` then `observer.close()`.
- [ ] `Orchestrator` is closed with `await orchestrator.close()` after use.
- [ ] `FastAPIChannel` uses `serve_forever()` (not `channel.run()` which does not exist).

### Architecture
- [ ] Observability is implemented via `EventBus` subscriptions — not by inserting print/log statements inside core logic.
- [ ] `BaseTool` subclasses do one thing. Side effects are documented in `description`.
- [ ] Config, storage, and tools are injected at construction time — not hard-coded inside classes.

### Code quality
- [ ] All public method signatures have type annotations (parameters + return type).
- [ ] `BaseTool.description` is written as an LLM-facing prompt: precise, unambiguous, no filler words.
- [ ] Comments explain **why** a decision was made, not **what** the code does.
- [ ] Private attributes (`_event_bus`, `_prompt_builder`) are accessed only when no public API exists. Add a comment noting the attribute is private and may change.

### Known private API surface (as of v0.1.0)
These APIs work but are not part of the public contract. They may change in a future release:
| Private attribute | Use case | Note |
|-------------------|----------|------|
| `agent._prompt_builder` | Register/activate dynamic skills | No public wrapper yet |
| `agent._event_bus` | Subscribe to events from outside the agent | Public alias: `agent.event_bus` (read-only) |
| `agent._hook_manager` | Direct HookManager access | Prefer `agent.hook()` / `@agent.on()` |

## Deep Reference

| Need | Where to look |
|------|--------------|
| Full API documentation | `/Users/neo/Desktop/project/git/neoagent/docs/guide.md` |
| Complete ROADMAP + release plan | `/Users/neo/Desktop/project/git/neoagent/docs/ROADMAP.md` |
| Source: core loop, prompt, compress | `/Users/neo/Desktop/project/git/neoagent/neoagent/core/` |
| Source: tools, BaseTool, registry | `/Users/neo/Desktop/project/git/neoagent/neoagent/tools/` |
| Source: multi-agent | `/Users/neo/Desktop/project/git/neoagent/neoagent/multi/` |
| Source: MCP client + transport | `/Users/neo/Desktop/project/git/neoagent/neoagent/mcp/` |
| Source: channels (FastAPIChannel) | `/Users/neo/Desktop/project/git/neoagent/neoagent/channels/` |
| Source: events, EventBus, hooks | `neoagent/events.py`, `neoagent/hooks.py` |
| Source: session, storage | `neoagent/session.py` |
| Source: memory system | `/Users/neo/Desktop/project/git/neoagent/neoagent/memory/` |
| Source: observer, eval | `neoagent/observe.py`, `neoagent/observe_subscriber.py`, `neoagent/eval/` |
| Architecture principles (why) | `wiki/` in ai-knowledge — use the `ai-knowledge` skill to navigate |
| Pattern cookbook | `cookbook/` in ai-knowledge — use the `ai-knowledge` skill to navigate |
| Install | `pip install -e /path/to/neoagent` (v0.1.0, not on PyPI) |
```

- [ ] Verify both the checklist section and deep reference table were appended. The checklist must have the 4 sub-sections (Correctness, Performance and resources, Architecture, Code quality). The reference table must have at least 12 rows.

---

## Task 9: Final verification + commit

**Goal:** Validate the skill file is complete, well-formed, and loadable, then commit.

- [ ] Check the final file has all required sections by verifying the following headings are present in order:
  1. `# neoagent SDK` (H1 title)
  2. `## Install`
  3. `## Quick Start`
  4. `## Module Architecture`
  5. `## wiki Module → neoagent Source`
  6. `## Scenario Quick Reference`
  7. `## Code Skeletons`
  8. `## Anti-patterns`
  9. `## Production-Ready Checklist`
  10. `## Deep Reference`

- [ ] Count all fenced code blocks (opening ` ``` ` lines). Verify each opening fence has a matching closing fence. If any fence is unclosed, fix it before continuing.

- [ ] Verify the frontmatter at the top of the file is valid YAML: it starts with `---`, ends with `---`, and contains both `name:` and `description:` fields.

- [ ] Verify the trigger `description` in frontmatter contains the required keywords: `neoagent SDK`, `memory`, `multi-agent`, `hooks`, `HTTP channel`.

- [ ] Run a quick smoke-check: confirm the file size is greater than 10 KB (the content should be substantial). Command: `wc -c ~/.claude/skills/neoagent/SKILL.md` — expect > 10000 bytes.

- [ ] Stage and commit the new skill file:
  ```bash
  cd /Users/neo/Desktop/project/git/ai-\ knowledge
  git add docs/superpowers/plans/2026-04-14-neoagent-sdk-skill.md
  git commit -m "feat(plans): add neoagent SDK skill implementation plan"
  ```

- [ ] After commit succeeds, print the file path of the created skill for confirmation:
  `~/.claude/skills/neoagent/SKILL.md`

---

## Summary

| Task | Output | Time estimate |
|------|--------|--------------|
| 1 — Scaffold | File created, frontmatter + overview | 2 min |
| 2 — Architecture diagram | 12-module text diagram | 3 min |
| 3 — wiki mapping table | 12-row source map | 2 min |
| 4 — Scenario table | 10-row quick reference | 2 min |
| 5 — Code skeletons 1–5 | 5 production-ready code blocks | 4 min |
| 6 — Code skeletons 6–10 | 5 production-ready code blocks | 4 min |
| 7 — Anti-patterns | 4 bad/good pairs | 3 min |
| 8 — Checklist + reference | Quality gates + path index | 3 min |
| 9 — Verify + commit | File validated, committed | 2 min |
| **Total** | | **~25 min** |

**Key design decisions captured in this plan:**
- `serve_forever()` is the correct FastAPIChannel method (not `run(host, port)` — that does not exist)
- `agent.resume(session_id)` is the preferred session recovery API (calls `Session.resume` internally via `JsonFileStorage`)
- Hook registration uses `agent.hook(type, handler)` / `@agent.on(type)` — NOT direct HookManager access
- `agent.event_bus` is a public property; `agent._event_bus` is the same object but the private name
- `EvalRunner.run(cases)` takes `list[EvalCase]`, each `EvalCase` has `messages: list[Message]` + `assertion: Callable[[str], bool]`
