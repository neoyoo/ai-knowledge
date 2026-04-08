# Hermes Agent Ingest Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deep-ingest Hermes Agent (Nous Research) into the AI Engineering Knowledge Base, producing 12 L2 pages and updating all 12 L1 comparison pages.

**Architecture:** Each L2 page requires independent source code analysis of hermes-agent, then L1 pages are batch-updated with new comparison columns and deepened design tradeoffs. Coverage scan validates completeness.

**Source:** `/Users/neo/Desktop/project/git/hermes-agent/` (Hermes Agent v0.8.0, by Nous Research)

**Key Source Files by Subsystem:**
- Agent Loop: `run_agent.py`, `model_tools.py`, `tools/registry.py`
- State: `hermes_state.py`, `agent/trajectory.py`
- Memory: `agent/memory_manager.py`, `tools/memory_tool.py`, `plugins/memory/`
- Skills: `tools/skills_hub.py`, `tools/skills_tool.py`, `agent/skill_utils.py`
- Prompt: `agent/prompt_builder.py`, `agent/context_references.py`
- Context: `agent/context_compressor.py`, `trajectory_compressor.py`
- Gateway: `gateway/run.py`, `gateway/session.py`, `gateway/platforms/`
- Tools: `tools/*.py`, `tools/registry.py`, `model_tools.py`
- CLI: `cli.py`, `hermes_cli/*.py`
- RL: `rl_cli.py`, `tools/rl_training_tool.py`, `environments/agent_loop.py`
- Cron: `cron/scheduler.py`, `cron/jobs.py`
- ACP: `acp_adapter/server.py`, `acp_adapter/session.py`
- MCP: `mcp_serve.py`, `tools/mcp_tool.py`

---

## Task 1: Create L2 Pages — Core Runtime (4 pages)

**Files to create:**
- `wiki/_impl/query-loop--hermes-agent.md`
- `wiki/_impl/runtime-state--hermes-agent.md`
- `wiki/_impl/prompt-system--hermes-agent.md`
- `wiki/_impl/context-management--hermes-agent.md`

Each page follows the established L2 format:
```
---
title: "<Concept> — Hermes Agent"
category: L2
parent: "[[<concept>]]"
source: hermes-agent
source_version: "0.8.0"
confidence: high
created: 2026-04-08
updated: 2026-04-08
---

## 概述
(One paragraph: what this subsystem does in Hermes Agent, its key design choice)

## 架构分析
(Multiple subsections with deep analysis of design patterns, data flow, state management)

## 关键代码路径
(File paths + key functions/classes with explanations)

## 设计亮点
(Bullet points: what's novel or well-designed compared to other projects)

## 局限性
(Bullet points: limitations, missing features, design tradeoffs that cost something)
```

- [ ] **Step 1.1:** Analyze `run_agent.py` (AIAgent class, run_conversation method, callback architecture, error recovery, iteration budget), `model_tools.py` (tool dispatch, async bridging) → write `query-loop--hermes-agent.md`
- [ ] **Step 1.2:** Analyze `hermes_state.py` (SQLite WAL, FTS5, schema versioning, cost tracking, session tables) → write `runtime-state--hermes-agent.md`
- [ ] **Step 1.3:** Analyze `agent/prompt_builder.py` (7-stage assembly pipeline, platform hints, skill injection, context file discovery, injection blocking), `agent/context_references.py` → write `prompt-system--hermes-agent.md`
- [ ] **Step 1.4:** Analyze `agent/context_compressor.py` (conversation-time compression, head/tail protection, iterative summary), `trajectory_compressor.py` (post-trajectory compression for training) → write `context-management--hermes-agent.md`

## Task 2: Create L2 Pages — Capabilities (3 pages)

**Files to create:**
- `wiki/_impl/tool-system--hermes-agent.md`
- `wiki/_impl/memory-system--hermes-agent.md`
- `wiki/_impl/multi-agent--hermes-agent.md`

- [ ] **Step 2.1:** Analyze `tools/registry.py` (singleton registry, registration format, availability checks), `model_tools.py` (discovery, dispatch, async bridging), `toolsets.py` (toolset groups, resolution), `toolset_distributions.py` (sampling for batch) → write `tool-system--hermes-agent.md`
- [ ] **Step 2.2:** Analyze `agent/memory_manager.py` (dual-layer design, nudge mechanism, context fencing), `tools/memory_tool.py`, `plugins/memory/` (8 providers, single external constraint) → write `memory-system--hermes-agent.md`
- [ ] **Step 2.3:** Analyze `tools/delegate_tool.py` (child agent spawning, restricted toolsets, shared iteration budget, max depth), parallel execution with ThreadPoolExecutor → write `multi-agent--hermes-agent.md`

## Task 3: Create L2 Pages — Extension (3 pages)

**Files to create:**
- `wiki/_impl/hooks--hermes-agent.md`
- `wiki/_impl/mcp-skills--hermes-agent.md`
- `wiki/_impl/channel-remote--hermes-agent.md`

- [ ] **Step 3.1:** Analyze callback architecture in `run_agent.py` (tool_progress, tool_complete, thinking, clarify, stream_delta, step callbacks), how CLI/gateway/ACP wire different callback implementations → write `hooks--hermes-agent.md`
- [ ] **Step 3.2:** Analyze `mcp_serve.py` (MCP server exposing conversations), `tools/mcp_tool.py` (MCP client), `tools/skills_hub.py` (Skills Hub, quarantine scanning, provenance), `agent/skill_utils.py` (FTS5 indexing, platform filtering), skill format (SKILL.md) → write `mcp-skills--hermes-agent.md`
- [ ] **Step 3.3:** Analyze `gateway/run.py` (async event loop, platform routing), `gateway/session.py` (SessionSource, reset policies), `gateway/platforms/` (17 adapters), `cron/` (scheduled delivery), PII redaction → write `channel-remote--hermes-agent.md`

## Task 4: Create L2 Pages — Reliability (2 pages)

**Files to create:**
- `wiki/_impl/session-recovery--hermes-agent.md`
- `wiki/_impl/evaluation-observability--hermes-agent.md`

- [ ] **Step 4.1:** Analyze session persistence in `hermes_state.py` (parent-child chains, compression-triggered splits), session resume in `cli.py`, gateway session continuity → write `session-recovery--hermes-agent.md`
- [ ] **Step 4.2:** Analyze cost tracking (billing providers, estimated vs actual), `batch_runner.py` (multiprocessing trajectory generation), `trajectory_compressor.py` (compression for RL training), `rl_cli.py` + `tinker-atropos/` (RL evaluation) → write `evaluation-observability--hermes-agent.md`

## Task 5: Update All 12 L1 Pages

**Files to modify:** All 12 L1 pages in `wiki/`

For each L1 page:
- [ ] **Step 5.1:** Add `hermes-agent` to `sources:` list in frontmatter
- [ ] **Step 5.2:** Add Hermes Agent column to comparison table (核心设计 | 关键特点 | 局限)
- [ ] **Step 5.3:** Add `[[<concept>--hermes-agent]]` to L2 详情 section
- [ ] **Step 5.4:** Deepen design tradeoffs based on Hermes Agent's novel approaches:
  - Query Loop: callback architecture as hook mechanism, shared iteration budgets
  - Tool System: registry pattern with availability checks, toolset distribution sampling
  - Prompt System: 7-stage pipeline with injection blocking, skin engine
  - Context Management: dual-phase compression (live + post-trajectory)
  - Memory System: pluggable provider ecosystem, nudge mechanism, context fencing
  - Multi-Agent: restricted child toolsets, depth-limited delegation
  - Runtime State: SQLite WAL + FTS5, schema versioning, cost billing modes
  - Session Recovery: compression-triggered session splitting, parent-child chains
  - Hooks: callback-based (vs shell/function hooks), per-callsite wiring
  - MCP & Skills: markdown skills with FTS5, Skills Hub with quarantine, MCP bidirectional
  - Channel & Remote: 17 platform adapters, gateway architecture, cron delivery
  - Evaluation: trajectory generation + compression for RL, cost tracking pipeline

## Task 6: Coverage Scan

- [ ] **Step 6.1:** List all top-level directories/modules in hermes-agent, confirm each is covered by at least one L2 page
- [ ] **Step 6.2:** Cross-check each L1 concept with keyword searches in hermes-agent source for uncovered implementations
- [ ] **Step 6.3:** Fix any gaps found

---

## Hermes Agent Novel Contributions (Value Justification)

| Concept | What Hermes Adds That Others Don't |
|---------|-----------------------------------|
| Query Loop | Callback architecture with 6 typed callbacks; fallback model activation; shared parent+child iteration budget |
| Tool System | Registry pattern with runtime availability checks; toolset distribution sampling for batch RL runs |
| Prompt System | 7-stage assembly pipeline; prompt injection blocking with pattern matching + unicode redaction; skin engine |
| Context Management | Dual-phase compression (live conversation + post-trajectory for training data); rough token estimation |
| Memory System | Pluggable provider ecosystem (8 backends); nudge mechanism for proactive saving; context fencing anti-injection |
| Multi-Agent | Depth-limited delegation with restricted child toolsets; shared iteration budget prevents runaway |
| Runtime State | SQLite WAL + FTS5 full-text search; schema versioning with auto-migration; dual cost tracking (estimated + actual) |
| Session Recovery | Compression-triggered session splitting with parent-child chains; cross-platform session continuity |
| Hooks | Pure callback architecture (vs shell/function hooks); per-callsite wiring (CLI vs gateway vs ACP vs RL) |
| MCP & Skills | Bidirectional MCP (server + client); markdown skills with FTS5 relevance scoring; Skills Hub with security quarantine |
| Channel & Remote | 17 platform adapters (largest coverage); gateway architecture with async routing; PII redaction; cron delivery |
| Evaluation | Batch trajectory generation; trajectory compression pipeline for RL training; cost tracking with billing providers |
