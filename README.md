# KEEP — Knowledge Engine for Engineering Persistence

[![skills.sh](https://skills.sh/b/CuriousDolphin/KEEP)](https://skills.sh/CuriousDolphin/KEEP)

A memory layer for agentic, spec-driven software development.

Code evolves; the reasoning behind it usually doesn't get written down. KEEP keeps a small, structured `/knowledge` directory next to the code — specs, ADRs, tagged operational and architecture notes, and an ideas inbox — and exposes **six** commands the agent uses to consult and maintain it as the codebase changes.

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

**Claude Code plugin** — adds the skill plus **six** real slash commands with autocomplete:

```text
/plugin marketplace add CuriousDolphin/KEEP
/plugin install keep@keep-knowledge
```

**skills.sh** — adds only the skill. Works in Claude Code, Cursor, Codex, and other Agent Skills compatible tools:

```bash
npx skills add CuriousDolphin/KEEP --skill keep -a claude-code -y
```

Both paths run the same logic — the skill itself is the source of truth. The plugin install registers `/keep-ask`, `/keep-compile`, … as deterministic slash commands; the skills.sh install relies on the skill triggering on either the command name or the equivalent natural language.

## Commands

Six slash commands (one root + five action verbs), plus a one-time bootstrap script.

| Step | Surface | Purpose | Writes? |
|---|---|---|---|
| Entry point | `/keep` | Dashboard. If `/knowledge` missing, offers to run `scripts/init.sh`. Otherwise prints counts + anchor coverage + hints. | No |
| Adoption (one-time) | `bash skills/KEEP/scripts/init.sh` (or via `/keep` on first run) | Scaffold `/knowledge`, scan pre-existing docs, append KEEP snippet to `AGENTS.md` / `CLAUDE.md` / `.cursorrules`. | Scaffold only |
| Read | `/keep-ask <question>` | Synthesize an answer from `/knowledge` with `[id]` citations. Say *"just list paths"* for list-only output (no synthesis) when you'll open the files yourself. | No |
| Write | `/keep-compile [source] [--dry]` | Two phases in one command — classify a diff/source, then apply updates and regenerate `INDEX.md` via `scripts/build_index.py`. `--dry` stops after classification. Suggests `anchors:` candidates when writing new specs. | Yes (unless `--dry`) |
| Enforce | `/keep-check-drift [source]` | **Deterministic** — `scripts/check_drift.py` verifies every anchor in spec frontmatter against current code (Go, Python, TypeScript). Exit 1 blocks merge. CI / pre-commit ready. | No |
| Hygiene | `/keep-govern` | Stale, duplicated, or contradicting knowledge; oversize files; `TODO(KEEP)` markers; draft ideas aging. Run weekly. | No |
| Capture | `/keep-idea <description>` | Capture a parked thought under `ideas/` with `type: idea`, `status: draft`. | Yes |

**Typical flows**

- **Questions** (`how`, `why`, `where`, `is it safe`): `/keep-ask` first — not ad-hoc grep from memory. If you only need paths (because you'll open the files yourself), tell `/keep-ask` "just list paths".
- **After meaningful code changes**: `/keep-compile` (single command, observe + write + regen). Add `--dry` if you want to preview without writing.
- **Before merge**: `/keep-check-drift` on the diff.
- **Parked ideas** (not implementing now): `/keep-idea "..."`.
- **Hygiene**: `/keep-govern` occasionally (weekly at most).

First-time adoption: `bash skills/KEEP/scripts/init.sh` then `/keep-compile ./docs/` for accepted ingestion candidates.

## Examples

**First-time adoption on an existing repo**

```text
$ bash skills/KEEP/scripts/init.sh
Scaffolds /knowledge/. Detects monorepo layout. Scans docs/, ARCHITECTURE.md,
notes/. Lists ingestion candidates. Appends KEEP workflow to AGENTS.md.
Nothing migrated until /keep-compile.

> /keep-compile ./docs/
Phase 1 (observe): proposes per-file target type (spec / ADR / runbook-tagged / architecture-tagged / idea / split).
Phase 2 (compile): migrates each accepted candidate with per-file approval. Adds provenance comments.
Regenerates INDEX.md from YAML frontmatter.
```

**During normal work**

```text
> /keep-ask how does JWT refresh work?
Reads INDEX.md, opens 2–3 specs/ADRs, answers with [SPEC-…] / [ADR-…] citations.

> /keep-ask auth flow — just list paths, I'll read them myself
Same filter logic, returns ids + paths + one-line descriptions (no synthesis).

[implement JWT refresh]

> /keep-compile --dry
Phase 1 only: classifies the diff (Feature + Decision). Surfaces nearby README as candidate. No writes.

> /keep-compile
Both phases: classify + write spec/ADR with valid frontmatter + rebuild INDEX.md.
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
commands/                 thin slash-command wrappers (six commands: /keep + five action verbs; bootstrap is scripts/init.sh)
skills/KEEP/              SKILL.md (source of truth) + references/ + scripts/
  scripts/build_index.py  regenerates /knowledge/INDEX.md from frontmatter
```

Full specification: [SKILL.md](skills/KEEP/SKILL.md). The files in `commands/` exist so plugin-installed users get real slash commands with autocomplete; the skill body behaves the same when triggered via slash command or natural language.

## Develop locally

```bash
claude plugin validate .
```
