# KEEP — Knowledge Engine for Engineering Persistence

[![skills.sh](https://skills.sh/b/CuriousDolphin/KEEP)](https://skills.sh/CuriousDolphin/KEEP)

A memory layer for agentic, spec-driven software development.

Code evolves; the reasoning behind it usually doesn't get written down. KEEP keeps a small, structured `/knowledge` directory next to the code — specs, ADRs, tagged operational and architecture notes, and an ideas inbox — and exposes seven slash commands the agent uses to consult and maintain it as the codebase changes. Spec claims are bound to specific code symbols via `anchors:`; a deterministic drift check fails CI when reality diverges from the spec.

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

**Claude Code plugin** — adds the skill plus seven real slash commands with autocomplete:

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

Seven slash commands. One root, one bootstrap, five action verbs.

| Step | Surface | Purpose | Writes? |
|---|---|---|---|
| Entry point | `/keep` | Dashboard. If `/knowledge` is missing, points the user to `/keep-init`. Otherwise prints counts + anchor coverage (spec-level and code-level) + adaptive hints. | No |
| Bootstrap | `/keep-init` | Scaffold `/knowledge`, install `SPEC-000-keep.md` (self-spec), append KEEP snippet to `AGENTS.md` / `CLAUDE.md` / `.cursorrules`. Always asks consent before writing. | Scaffold only |
| Read | `/keep-ask <question>` | Synthesize an answer from `/knowledge` with `[id]` citations. Say *"just list paths"* for list-only output. | No |
| Write | `/keep-compile [source] [--dry]` | One command, two phases — classify a source, write/update specs+ADRs, propose anchors from concrete values, regenerate `INDEX.md`. `--dry` stops after classification. When the source is pre-existing markdown (not a git diff), the agent asks before writing. | Yes (unless `--dry`) |
| Enforce | `/keep-check-drift [source]` | **Deterministic** — `scripts/check_drift.py` verifies every anchor in spec frontmatter against current code (Go, Python, TypeScript). Exit 1 blocks merge. CI / pre-commit ready. | No |
| Capture | `/keep-idea <description>` | Capture a parked thought under `ideas/` with `type: idea`, `status: draft`. Searches for prior art first. | Yes |
| Hygiene | `/keep-govern` | Stale, duplicated, or contradicting knowledge; oversize files; `TODO(KEEP)` markers; aging draft ideas. Run weekly at most. | No |

**Typical flows**

- **Questions** (`how`, `why`, `where`, `is it safe`): `/keep-ask` first — not ad-hoc grep. If you only need paths, tell `/keep-ask` "just list paths".
- **First-time adoption**: `/keep-init`. The agent asks consent, scaffolds `/knowledge`, installs `SPEC-000-keep.md`, appends the AGENTS.md snippet.
- **Pre-existing docs**: `/keep-compile ./docs/` (folder) or `/keep-compile docs/auth/jwt.md` (file). The agent asks: cordon (safe default) or migrate-with-verification.
- **After meaningful code changes**: `/keep-compile` (single command, classify + propose anchors + write + regen). Add `--dry` to preview.
- **Before merge**: `/keep-check-drift`. Exit 1 blocks merge.
- **Parked ideas** (not implementing now): `/keep-idea "..."`.
- **Hygiene**: `/keep-govern` occasionally (weekly at most).

## Examples

**First-time adoption**

```text
> /keep-init
Asks: "Scaffold /knowledge, install SPEC-000-keep, append AGENTS.md snippet? [y/N]"
On y → creates /knowledge/{docs/{specs,decisions},ideas}, INDEX.md,
       /knowledge/docs/specs/keep/SPEC-000-keep.md, appends AGENTS.md.
       Mentions optional CI/pre-commit setup, does not auto-install.
```

**Handling pre-existing docs**

```text
> /keep-compile ./docs/
"./docs/ has 12 markdown files. Two options:
  (a) Cordon the folder — one ADR declares it out of scope.        ← default
  (b) Walk file-by-file — ask per file: migrate, cordon, or skip.
Which do you want?"

> /keep-compile docs/auth/jwt.md
"Two options:
  (a) Migrate — extract claims, verify each against current code,
                ask about disagreements, write a clean spec with anchors.
  (b) Cordon — mark as legacy / out of scope.                       ← default
Which do you want?"
```

**During normal work**

```text
> /keep-ask how does JWT refresh work?
Reads INDEX.md, opens 2–3 specs/ADRs, answers with [SPEC-…] / [ADR-…] citations.

> /keep-ask auth flow — just list paths, I'll read them myself
Same filter logic, returns ids + paths + one-line descriptions (no synthesis).

[implement JWT refresh]

> /keep-compile --dry
Phase 1 only: classifies the diff, detects any renames, lists suggested updates. No writes.

> /keep-compile
Both phases: classify + write spec/ADR with valid frontmatter + propose anchors
            from concrete values in the diff + regen INDEX.md (Backlinks section refreshed).
```

**Before merge**

```text
> /keep-check-drift
Deterministic anchor verification. Exit code 1 blocks merge if any anchor
fails to bind to its current code symbol/value/signature.
```

**Park an idea**

```text
> /keep-idea "explore edge-signed tokens for service-to-service; not this quarter"
Searches /knowledge for prior art, then writes ideas/YYYY-MM-DD-slug.md in draft.
Promotion to spec/ADR happens later via /keep-compile with explicit approval.
```

**Periodic hygiene**

```text
> /keep-govern
Surfaces contradicting ADRs, oversize specs, aging draft ideas, TODO(KEEP) markers.
Suggestions only — never writes without approval.
```

## Design principles

- **Memory, not narration.** Capture rationale, edge cases, rejected alternatives. Skip implementation walkthroughs that just re-narrate the code.
- **Brownfield-first.** Grow from real code changes. For pre-existing docs the agent asks, then either cordons (safe default) or migrates per file with a verify-against-code pass — no silent imports.
- **YAML frontmatter on every durable file** under `docs/` and `ideas/` so `/keep-ask`, the index, and `/keep-check-drift` can consume and enforce links. See [references/file_formats.md](skills/KEEP/references/file_formats.md).
- **INDEX.md is derived** — regenerated from frontmatter (including the auto-built Backlinks section) by `skills/KEEP/scripts/build_index.py`. Hand-editing it reintroduces drift.
- **Ask before inventing.** When `/keep-compile` would write a high-stakes field the diff cannot establish, it asks in batch mode (structured multiple-choice with a default) instead of guessing.
- **Minimal diffs.** Updates preserve human-written rationale verbatim. Never rewrite whole files.
- **Self-spec'd.** `/keep-init` installs `SPEC-000-keep.md` inside the knowledge layer so KEEP's conventions are documented *in* the repo, not just *about* it.

## Repository layout

```
.claude-plugin/                manifest
commands/                      slash-command wrappers (/keep, /keep-init, /keep-ask, /keep-compile,
                                /keep-check-drift, /keep-idea, /keep-govern)
skills/KEEP/
  SKILL.md                     source of truth
  references/                  file_formats, brownfield, monorepo, setup, templates/SPEC-000-keep.md
  scripts/
    init.sh                    one-time bootstrap (called by /keep-init)
    status.py                  /keep dashboard probe
    build_index.py             regenerates /knowledge/INDEX.md from frontmatter + backlinks
    check_drift.py             deterministic anchor verification (CI/pre-commit ready)
    coverage.py                code-level anchor coverage report
```

Full specification: [SKILL.md](skills/KEEP/SKILL.md). The files in `commands/` exist so plugin-installed users get real slash commands with autocomplete; the skill body behaves the same when triggered via slash command or natural language.

---

## Inspiration

KEEP wouldn't exist in this shape without two ideas it stands on the shoulders of:

- **[Andrej Karpathy's LLM Wiki](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)** ([original tweet](https://x.com/karpathy/status/2040470801506541998)). The "personal wiki the LLM keeps up to date for you" pattern. KEEP is what happens when you take that idea and apply it specifically to a software repository — same compile-and-cross-link instinct, but tuned for *intent* (specs, ADRs, runbooks) rather than personal notes, and with an enforcement layer (drift checks against code anchors) that a personal wiki doesn't need.
- **[Anchored Dev](https://anchored-dev.org/getting-started/)**. The discipline of anchoring specifications to specific places in the code, so the spec and the implementation can be mechanically checked for agreement. `/keep-check-drift` and the `anchors:` block in spec frontmatter are KEEP's expression of this: when a spec claims to govern a constant, function, or test, it has to *point at it* — and a rename or deletion deterministically breaks the build until the spec is updated too.

The core insight from combining the two: a wiki the LLM keeps up to date is great, but a wiki the LLM keeps up to date *and that can fail CI when it lies* is what makes the knowledge layer reliable enough to trust long-term.

---