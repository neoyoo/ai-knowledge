# ai-knowledge

> A structured Obsidian vault and ontology for AI engineering — organized around concepts, not sources.

[English] | [中文](README.zh-CN.md)

## What it is

ai-knowledge is a personal knowledge base covering the AI engineering landscape: agent architecture, prompt engineering, tool systems, context management, multi-agent orchestration, MCP, evaluation, finetuning, and more. It is organized as an Obsidian vault with a formal ontology — 14 concept dimensions that cut across sources, each backed by L1 concept pages, L2 per-source implementation analyses, cross-concept pattern pages, and a lightweight ideas inbox.

The vault is kept public for transparency. It is not a turn-key product or a curated reading list. The structure reflects one engineer's workflow for making sense of a fast-moving field, retaining design decisions, and not re-researching the same questions twice.

The companion `kb-ingest` skill automates ingesting new source projects: deep source-code analysis, L2 page generation, L1 semantic patching, coverage scanning — all staged in `drafts/` before any wiki change is committed.

## Design Philosophy

**Knowledge is a network, not a filing cabinet.** Pages are connected by typed relations (`supports`, `contradicts`, `evolved_into`, `depends_on`). Isolated facts are cheap; cross-concept connections are where the value lives.

**Organize by concept dimension, not by source.** A page about context compression covers every relevant project in one place. Knowing "how neoagent does it" and "how Claude Code does it" side-by-side is more useful than having two separate project folders.

**Friction logging drives self-evolution.** Every time the KB fails to answer a question — content missing, too shallow, or off-target — the miss is recorded in `projects/<proj>/kb-friction.md`. Accumulated friction entries are the KB's honest signal for where to invest next. Without this, a knowledge base quietly rots while appearing healthy.

**Deterministic executor, not free-form LLM edits.** The ingest pipeline produces structured intent operations (`KEEP / UPDATE / MERGE / SUPERSEDE / ARCHIVE`). A deterministic executor applies them. LLMs propose; the executor commits. This prevents hallucination drift from corrupting the wiki over time.

**Ideas have a lifecycle.** Fleeting observations go into `ideas/` with `status: inbox`. They incubate, get promoted to `wiki/_patterns/` or `wiki/_insights/`, or are explicitly marked dead with a recorded reason. Nothing disappears silently.

## Quick Start

The primary interface is the `ai-knowledge` Claude Code skill. It routes queries to the right pages, enforces "check KB first" discipline, and records friction automatically.

```bash
# Symlink the skill into Claude Code
ln -s /path/to/ai-knowledge/skills/ai-knowledge ~/.claude/skills/ai-knowledge
```

Once enabled, Claude Code will consult the vault before doing any external research on agent architecture, design decisions, or cross-project comparisons. Example trigger phrases:

- "How do I design the context compression strategy?"
- "Compare how different projects handle session recovery."
- "Is there a pattern for combining free/recall with tool metadata?"

To ingest a new source project into the KB, use the `kb-ingest` skill (see `skills/kb-ingest/SKILL.md`).

## Structure

```
ai-knowledge/
├── wiki/               L1 concept pages + L2 per-source analyses
│   ├── *.md            14 L1 concept pages (query-loop, tool-system, context-management, …)
│   ├── _impl/          L2 pages: <concept>--<source>.md
│   ├── _insights/      Single-source design highlights worth extracting
│   └── _patterns/      Cross-concept combination patterns (transferable architecture vocabulary)
├── cookbook/           "How to do it" layer (prompts / tools / skills × patterns / templates / anti-patterns)
├── ideas/              Ideas inbox — status: inbox | incubating | promoted | dead
├── projects/           Per-project decision logs and friction journals
├── raw/                Symlinks to source project checkouts (read-only, never modified)
├── schema/             Ontology, lint rules, ingest strategies, page templates
├── shelf/              Archived sources — full analysis preserved, not merged into wiki
├── drafts/             In-progress ingest output (staged before review)
└── docs/               KB meta-docs (sdk-kb-alignment, etc.)
```

### Ontology

The schema defines 14 concept dimensions:

| Dimension | Covers |
|---|---|
| `query-loop` | Agent main loop, state machine, turns |
| `prompt-system` | Dynamic prompt assembly, sections, priorities |
| `tool-system` | Tool dispatch, registry, permission model |
| `context-management` | Context window, compression, free/recall |
| `memory-system` | Cross-session persistent memory |
| `runtime-state` | Session state, lifecycle, checkpointing |
| `session-recovery` | Resume, crash recovery, fault tolerance |
| `multi-agent` | Orchestrator/worker patterns, task delegation |
| `hooks` | Event hooks, lifecycle interception |
| `mcp-skills` | MCP protocol, skill extension points |
| `channel-remote` | HTTP channels, SSE streaming, FastAPI |
| `evaluation-observability` | Eval harnesses, metrics, tracing |
| `finetuning-system` | SFT, DPO, GRPO, training pipelines |
| `agent-registry-discovery` | Agent registry, service discovery, AgentCard |

Full definitions and typed relation types are in `schema/ontology.yaml`.

### Knowledge flow

```
raw/          →  ingest  →  drafts/   →  review  →  wiki/        (main trunk)
(immutable)                (staging)               wiki/_impl/   (L2 analyses)
                                                   wiki/_insights/
                                                   shelf/         (not merged, preserved)

ideas/                                 →  wiki/_patterns/
(inbox → incubating → promoted/dead)     wiki/_insights/
                                         cookbook/

projects/<proj>/kb-friction.md   →  drives next ingest priorities
```

## How to Use with kb-ingest

The `kb-ingest` skill automates adding a new source project. It runs 7 phases: pre-analysis, parallel L2 page writing (up to 12 concurrent subagents), L1 semantic patch generation, coverage scan (target ≥ 80%), tiered review, report generation, and self-optimization feedback. All output lands in `drafts/{source}/` and never touches `wiki/` directly until reviewed.

```bash
ln -s /path/to/ai-knowledge/skills/kb-ingest ~/.claude/skills/kb-ingest
```

Then in Claude Code: `"Ingest /path/to/some-project into the knowledge base"`.

## Companion Skills

Two skills ship with this repo:

- `skills/ai-knowledge/SKILL.md` — the read path. Query the vault, find relevant pages, record friction. Canonical guide for day-to-day use.
- `skills/kb-ingest/SKILL.md` — the write path. Autonomous 7-phase ingest pipeline. Full protocol for adding a new source project.

Symlink or copy both to `~/.claude/skills/` to enable them in Claude Code.

## Acknowledgments

- **[Obsidian](https://obsidian.md)** — the vault substrate. Markdown + backlinks + local-first storage are what make this kind of knowledge graph practical at all.
- **Niklas Luhmann's Zettelkasten method** — concept-first organization (cut by dimension, not by source); typed relations between cards; ideas have a lifecycle from inbox to archive.
- **Andrej Karpathy** — the framing that defines the field this vault tries to map: LLMs as a new compute surface, "Software 3.0," and the recurring discipline of re-deriving rather than re-googling. Many wiki pages started from a question Karpathy posed.
- **[Claude Code](https://claude.com/claude-code)** — the skills system as a knowledge-loading interface, mirrored in `kb-ingest`'s 7-phase pipeline (drafts → review → wiki).

Every wiki page traces to specific source projects. Every cross-concept pattern came from reading those sources together.

## License

MIT. See `LICENSE`.

## Status

Personal knowledge base, kept public for transparency. Not intended as a turn-key product — the ontology and patterns reflect the author's specific workflow for AI agent development. Content quality varies by source: `wiki/` pages are mature; `shelf/` entries are archived with known limitations noted.
