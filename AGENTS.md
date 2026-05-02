# ai-knowledge Project Instructions

This repository is an AI engineering knowledge base. Treat it as a decision-support system, not a generic note dump.

## Repository Purpose

- `wiki/` is the formal knowledge spine: L1 concept pages, L2 per-source implementation analysis, cross-concept patterns, and single-source insights.
- `cookbook/` is the practical layer: patterns, templates, anti-patterns, and reusable implementation guidance.
- `ideas/` is the inbox for not-yet-promoted thoughts.
- `projects/` records real project decisions and KB friction.
- `shelf/` archives sources that are useful but not mature enough for the main wiki.
- `docs/` is for KB operations and meta-documentation.

Read `CLAUDE.md` before making structural changes to the knowledge base. It is the current repo-level governance spec.

## Editing Rules

- Keep changes scoped to the user's request.
- Do not modify `wiki/`, `skills/`, `cookbook/`, `ideas/`, `shelf/`, or `projects/` unless the task explicitly calls for it.
- Do not run git commands unless the user asks.
- Use `rg` / `rg --files` for searches.
- Preserve existing Chinese/English style in the file being edited.
- Do not move content between layers casually. If a page belongs in another layer, say so and explain why.

## Knowledge Quality Rules

- Tag claims as source-backed when possible. L1 claims should trace to L2 pages, source files, or documented project decisions.
- Fix contradictions before adding new derived content.
- Prefer links that improve navigation and conceptual clarity, not links added only to make the graph denser.
- For missing files referenced by a task, write "file not found" and continue.
- For `.patch` files, treat them as generated/reference material unless the task asks to apply them.

## Wiki Conventions

- L1 pages live at `wiki/<concept>.md`.
- L2 pages live at `wiki/_impl/<concept>--<source>.md`.
- Cross-concept transferable architecture patterns live at `wiki/_patterns/<slug>.md`.
- Single-source notable designs live at `wiki/_insights/<source>--<design>.md`.
- A pattern must be cross-concept, transferable, backed by at least one reference implementation, and solve a concrete design problem.

## Review Reports

When producing a KB review:

- Write findings first, grouped by requested area.
- Include file and line references whenever available.
- Mark findings with `[verified]` when read directly and `[inferred]` when reasoned from partial evidence.
- Keep each finding short and actionable.
- Do not modify files outside the requested report path.

## Codex Behavior In This Repo

- Be conservative with edits.
- Prefer small, checkable changes over broad rewrites.
- If a task uncovers governance debt, record it in the requested report or propose a separate branch; do not silently fix unrelated areas.
- When the user asks for optimization planning, split work into independently reviewable branches.
