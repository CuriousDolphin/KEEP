# KEEP — Knowledge Engine for Engineering Persistence

[![skills.sh](https://skills.sh/b/CuriousDolphin/KEEP)](https://skills.sh/CuriousDolphin/KEEP)

A memory layer for agentic, spec-driven software development.

Code evolves; the reasoning behind it usually doesn't get written down. KEEP keeps a small, structured `/knowledge` directory next to the code — specs, ADRs, tagged operational and architecture notes, and an ideas inbox — and exposes **seven** commands the agent uses to consult and maintain it as the codebase changes.

```
/knowledge
├── docs/
│   ├── specs/          what the system should do (behavioral specs)
│   │                     use tags `runbook` / `architecture` for ops and topology
│   └── decisions/      why it is the way it is (ADRs, flat ADR-NNNN-slug.md)
├── ideas/              half-formed proposals (promoted later via compile)
└── INDEX.md            navigation map — auto-generated, do not edit by hand
```

Runbooks and architecture live as **specs with reserved tags**, not separate top-level folders. Execution state belongs in your tracker or agent task lists, not under `/knowledge`.

## Install

**Claude Code plugin** — adds the skill plus **seven** real slash commands with autocomplete:

```text
/plugin marketplace add CuriousDolphin/KEEP
/plugin install keep@keep-knowledge
```

**skills.sh** — adds only the skill. Works in Claude Code, Cursor, Codex, and other Agent Skills compatible tools:

```bash
npx skills add CuriousDolphin/KEEP --skill keep -a claude-code -y
```

Both paths run the same logic — the skill itself is the source of truth. The plugin install registers `/keep-init`, `/keep-ask`, … as deterministic slash commands; the skills.sh install relies on the skill triggering on either the command name (`/keep-observe`) or the equivalent natural language ("osserva il diff e proponi gli update").

## Commands

| Command | Purpose | Writes? |
|---|---|---|
| `/keep-init` | Scaffold `/knowledge`, scan pre-existing docs, append KEEP snippet to `AGENTS.md` / `CLAUDE.md` / `.cursorrules` | No |
| `/keep-ask <question>` | **The read command** — synthesize an answer from `/knowledge` with `[id]` citations. Say *"just list paths"* for list-only output (no synthesis) when you're about to open the files yourself. | No |
| `/keep-observe [source]` | Classify impact of changes (git diff, PR, folder of docs) | No |
| `/keep-compile` | Apply suggested updates, migrate ingestion candidates, **regenerate `INDEX.md`** via `skills/KEEP/scripts/build_index.py` | Yes |
| `/keep-check-drift [source]` | Drift between diff and knowledge (`related:` patterns); CI / pre-commit friendly (exit 1 on drift) | No |
| `/keep-govern` | Stale, duplicated, or contradicting knowledge; oversize files; `TODO(KEEP)`; draft ideas aging | No |
| `/keep-idea <description>` | Capture a parked thought under `ideas/` with `type: idea`, `status: draft` | Yes |

**Typical flows**

- **Questions** (`how`, `why`, `where`, `is it safe`): `/keep-ask` first — not ad-hoc grep from memory. If you only need paths (because you'll open the files yourself), tell `/keep-ask` "just list paths".
- **After meaningful code changes**: `/keep-observe` → `/keep-compile` (compile applies updates and refreshes the index).
- **Before merge**: `/keep-check-drift` on the diff.
- **Parked ideas** (not implementing now): `/keep-idea "..."`.
- **Hygiene**: `/keep-govern` occasionally (for example weekly).

First-time adoption: `/keep-init` then `/keep-compile` for accepted ingestion candidates.

## Examples

**First-time adoption on an existing repo**

```text
> /keep-init
Scaffolds /knowledge/. Detects monorepo layout. Scans docs/, ARCHITECTURE.md,
notes/. Proposes ingestion candidates. Appends KEEP workflow to AGENTS.md.
Nothing migrated until compile.

> /keep-compile
Migrates each accepted candidate one by one. Runs elicitation where needed.
Regenerates INDEX.md from YAML frontmatter (build_index.py). Nothing hand-edited in the index.
```

**During normal work**

```text
> /keep-ask how does JWT refresh work?
Reads INDEX.md, opens 2–3 specs/ADRs, answers with [SPEC-…] / [ADR-…] citations.

> /keep-ask auth flow — just list paths, I'll read them myself
Same filter logic, returns ids + paths + one-line descriptions (no synthesis).

[implement JWT refresh]

> /keep-observe
Classifies the diff (Feature + Decision). Quotes commit message as ADR evidence.
Surfaces nearby README as ingestion candidate.

> /keep-compile
Creates or updates ADR/spec files with valid frontmatter, then rebuilds INDEX.md.
```

**Before merge**

```text
> /keep-check-drift
Linter-style report on behavioral / decisional / operational drift vs related: patterns.
Exit code 1 if merge should wait on knowledge updates.
```

**Park an idea**

```text
> /keep-idea "explore edge-signed tokens for service-to-service; not this quarter"
Writes ideas/YYYY-MM-DD-slug.md in draft; promotion happens later via compile with approval.
```

**Periodic hygiene**

```text
> /keep-govern
Surfaces contradicting ADRs, oversize specs, old draft ideas, TODO(KEEP) markers. Suggestions only.
```

## Design principles

- **Memory, not narration.** Capture rationale, edge cases, rejected alternatives. Skip implementation walkthroughs that just re-narrate the code.
- **Brownfield-first.** Grow from real changes or ingestion of existing docs. No retroactive backfill of the whole codebase.
- **YAML frontmatter on every durable file** under `docs/` and `ideas/` so `/keep-ask`, the index, and `/keep-check-drift` can consume and enforce links. See [references/file_formats.md](skills/KEEP/references/file_formats.md).
- **INDEX.md is derived** — regenerated from frontmatter by `skills/KEEP/scripts/build_index.py`. Hand-editing it reintroduces drift.
- **Ask before inventing.** When `/keep-compile` would write a high-stakes field the diff cannot establish, it asks in batch mode instead of guessing.
- **Minimal diffs.** Updates preserve human-written rationale verbatim. Never rewrite whole files.

## Repository layout

```
.claude-plugin/           plugin and marketplace manifest
commands/                 thin slash-command wrappers → skill steps (seven commands)
skills/KEEP/              SKILL.md (source of truth) + references/ + scripts/
  scripts/build_index.py  regenerates /knowledge/INDEX.md from frontmatter
```

Full specification: [SKILL.md](skills/KEEP/SKILL.md). The files in `commands/` exist so plugin-installed users get real slash commands with autocomplete; the skill body behaves the same when triggered via slash command or natural language.

## Develop locally

```bash
claude plugin validate .
```
