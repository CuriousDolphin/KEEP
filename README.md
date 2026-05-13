# KEEP — Knowledge Engine for Engineering Persistence

[![skills.sh](https://skills.sh/b/CuriousDolphin/KEEP)](https://skills.sh/CuriousDolphin/KEEP)

A memory layer for agentic, spec-driven software development.

Code evolves; the reasoning behind it usually doesn't get written down. KEEP keeps a small, structured `/knowledge` directory next to the code — specs, decisions (ADRs), architecture notes, runbooks — and exposes five commands the agent uses to maintain it as the codebase changes.

```
/knowledge
├── docs/
│   ├── specs/          what the system should do
│   ├── decisions/      why it is the way it is (ADRs)
│   ├── architecture/   topology, components, flows
│   └── runbooks/       symptoms, causes, mitigations
├── tasks/              ephemeral execution state
└── INDEX.md            navigation map
```

## Install

**Claude Code plugin** — adds the skill plus five real slash commands with autocomplete:

```text
/plugin marketplace add CuriousDolphin/KEEP
/plugin install keep@keep-knowledge
```

**skills.sh** — adds only the skill. Works in Claude Code, Cursor, Codex, and other Agent Skills compatible tools:

```bash
npx skills add CuriousDolphin/KEEP --skill keep -a claude-code -y
```

Both paths run the same logic — the skill itself is the source of truth. The plugin install registers `/keep-init`, `/keep-observe`, … as deterministic slash commands; the skills.sh install relies on the skill triggering on either the command name (`/keep-observe`) or the equivalent natural language ("osserva il diff e proponi gli update").

## Commands

| Command | Purpose | Writes? |
|---|---|---|
| `/keep-init` | Scaffold `/knowledge` and propose ingestion of pre-existing docs | No |
| `/keep-retrieve <topic>` | Load focused context for a topic | No |
| `/keep-observe [source]` | Classify the impact of changes (git diff, PR, folder of docs) | No |
| `/keep-compile` | Apply suggested updates and migrate ingestion candidates | Yes |
| `/keep-govern` | Surface stale, duplicated, or contradicting knowledge | No |

The standard loop is `retrieve → code → observe → compile`. `init` runs once at adoption; `govern` periodically.

## Examples

**First-time adoption on an existing repo**

```text
> /keep-init
Scaffolds /knowledge/. Detects 6 monorepo packages. Scans docs/, ARCHITECTURE.md,
notes/. Proposes 8 ingestion candidates (3 specs, 1 architecture, 2 runbooks,
1 ADR needing elicitation, 1 mixed-content to split). Nothing written yet.

> /keep-compile
Migrates each accepted candidate one by one. For the ADR, asks 3 batched questions
about rejected alternatives. Adds provenance comments. Updates INDEX.md.
```

**During normal work**

```text
> /keep-retrieve auth flow
Returns 4 paths grouped by Architecture / Decisions / Specs / Runbooks.

[implement JWT refresh]

> /keep-observe
Classifies the diff (Feature + Decision). Quotes the commit message
"switched from KServe because of CRD complexity" as ADR evidence.
Also surfaces ./services/auth/README.md as an ingestion candidate.

> /keep-compile
Creates ADR-0007, updates specs/auth/jwt.md, migrates the README into
specs/auth/overview.md. Prints a 6-line summary.
```

**Periodic hygiene**

```text
> /keep-govern
Surfaces 2 ADRs that contradict without a supersedes link, 1 spec >400 lines
to split, 3 task files older than 30 days. Suggestions only — waits for approval.
```

## Design principles

- **Memory, not narration.** Capture rationale, edge cases, rejected alternatives. Skip implementation walkthroughs that just re-narrate the code.
- **Brownfield-first.** Grow from real changes or ingestion of existing docs. No retroactive backfill of the whole codebase.
- **Ask before inventing.** When `/keep-compile` would write a high-stakes field the diff can't establish (rejected alternatives, runbook causes, edge cases), it asks instead of guessing.
- **Minimal diffs.** Updates preserve human-written rationale verbatim. Never rewrite whole files.

## Repository layout

```
.claude-plugin/    plugin and marketplace manifest
commands/          slash command wrappers (used by the Claude Code plugin install)
skills/KEEP/       the skill — SKILL.md (source of truth) + references/
```

Full specification: [SKILL.md](skills/KEEP/SKILL.md). The files in `commands/` are thin wrappers that route each slash command to the corresponding step of the skill — they exist so plugin-installed users get real slash commands with autocomplete. The skill body works identically whether triggered via slash command or natural language.

## Develop locally

```bash
claude plugin validate .
```
