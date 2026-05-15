# Setting up KEEP in an existing repository

KEEP is *brownfield-first*. Adoption is meant to take an hour, not a week. There is no upfront documentation requirement; knowledge grows from real changes from the moment KEEP is in place.

The recommended entry point is **`/keep-init`** — it scaffolds the layout, detects monorepo shape, scans the repo for pre-existing docs, and produces an ingestion proposal in one pass. The manual steps below remain useful for understanding what `/keep-init` does and for environments where the slash command isn't installed.

## Step 1 — scaffold the skeleton

If you're running the command, `/keep-init` does this for you. The manual equivalent, from the repo root:

```bash
mkdir -p knowledge/docs/specs
mkdir -p knowledge/docs/decisions
mkdir -p knowledge/docs/architecture
mkdir -p knowledge/docs/runbooks
mkdir -p knowledge/tasks
touch knowledge/INDEX.md
```

Empty subdirectories are fine. Do **not** generate retroactive specs, ADRs, or architecture docs for the existing code. The first real updates will come from the next meaningful change via `/keep-observe` and `/keep-compile`.

### Monorepo setup

KEEP uses a single `/knowledge/` directory at the repository root, even for monorepos. There are no per-package knowledge directories. Inside the single zone, `specs/` and `runbooks/` get a per-package subdirectory; `decisions/` and `architecture/` stay flat. See the **Monorepo support** section in the main SKILL.md for details.

`/keep-init` auto-detects monorepo shape (`pnpm-workspace.yaml`, `package.json` workspaces, Cargo workspaces, `go.work`, top-level `apps/` `packages/` `services/`) and asks you to confirm the package list before scaffolding per-package subdirs. The manual equivalent for known packages:

```bash
mkdir -p knowledge/docs/specs/auth-service
mkdir -p knowledge/docs/specs/inference-api
mkdir -p knowledge/docs/specs/shared
mkdir -p knowledge/docs/runbooks/auth-service
mkdir -p knowledge/docs/runbooks/shared
```

Empty subdirectories are still fine — they just establish the layout convention.

### Ingesting pre-existing documentation

If the repo already has docs scattered around (`README.md`, `docs/`, `ARCHITECTURE.md`, `notes/`, wiki exports), do **not** copy them into `/knowledge/docs/` by hand. Use the brownfield ingestion workflow:

1. **`/keep-init`** scans the canonical doc locations and produces a classification proposal — no writes. (Or, for a specific folder at any later time: `/keep-observe ./old-docs/`.)
2. **Review the proposal.** Accept, correct, split, or drop each candidate.
3. **`/keep-compile`** migrates the accepted files one at a time, with elicitation for missing high-stakes fields and verbatim quoting of source content. Each migrated file gets a provenance comment.

This produces structured, cross-linked knowledge files instead of a dump of legacy markdown. The full heuristic catalog and worked examples are in `references/brownfield.md`.

## Step 2 — tell agents how to use KEEP

`/keep-init` appends the canonical snippet to whichever instruction file already exists at the repo root (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`). If none exists, it asks which to create.

**The single source of truth for the snippet is the `## AGENTS.md snippet — install in the repo` section of `SKILL.md`.** Do not maintain a second copy here — the snippet drifts the moment two sources exist. `/keep-init` reads that section verbatim and appends it.

The snippet is intentionally hard:

- It frames `/keep-ask` as **mandatory** consultation before answering questions about behavior, design, history, or any uncertainty about conventions. The wording "Before answering ANY of these, run `/keep-ask <topic>` first" is the hammer that defeats the dominant failure mode (under-triggering on the read path).
- It instructs the agent to **say so explicitly** when `/keep-ask` returns no indexed knowledge, rather than falling back to generic knowledge presented as repo truth.
- It marks code changes that touch behavior/architecture/operations as **incomplete** until `/keep-observe` → `/keep-compile` has run. That last sentence is the one that does most of the work — without it, knowledge updates get skipped under deadline pressure.
- It wires `/keep-check-drift` into the merge path (`exit-code 1 blocks merge`), and `/keep-govern` to weekly hygiene.
- It carves out `/keep-idea` for half-formed thoughts so they don't get lost in chat.

When auditing an adoption, the failure mode to look for is a softened snippet: someone removed the "mandatory" wording, or replaced "ANY of these" with "some of these", and the agent quietly stopped consulting `/knowledge` on questions. Restore the canonical version from `SKILL.md` and the read path comes back to life.

## Step 3 — optional starting content

If the team already has a few obvious decisions worth capturing (the kind of thing that comes up in every onboarding conversation — "why are we on Ray Serve?", "why do we use Auth0?"), write one or two ADRs by hand on day one. This gives `/keep-ask` something substantive to return on the first question and sets the tone for the format.

Do **not** try to write more than three or four. The point is to seed, not to backfill.

## Step 4 — verify the loop works

On the next real code change, run the full cycle:

```
/keep-ask <topic>           # synthesize prior context, OR ask for list-only paths
[make the change]
/keep-observe               # classify impact (+ nearby docs)
/keep-compile               # apply updates (with batch elicitation if needed)
```

If `/keep-compile` produces a sensible diff on real work and asks the right kind of question for ADRs/new specs, KEEP is set up correctly. If it produces nothing or produces too much, recalibrate: the command contract is narrow on purpose.

## Common adoption mistakes

- **Backfilling.** Trying to document the existing codebase before using KEEP. Don't. Document as code changes — or use `/keep-init` and brownfield ingestion if you genuinely have existing docs to migrate.
- **Skipping `/keep-observe`.** Going straight to `/keep-compile` produces lower-quality updates because there is no classification step to anchor the changes.
- **Treating KEEP as a task tracker.** KEEP only stores durable knowledge (specs, ADRs, ideas). Active work-in-progress belongs in your ticket tracker or agent task list. If a working note turns out to contain durable signal (a decision, a runbook step, a confirmed edge case), capture it via `/keep-observe` → `/keep-compile`, not by dumping the note into `/knowledge`.
- **Running `/keep-govern` every cycle.** It is meant to be periodic — weekly at most. Running it constantly creates noise.
- **Multiple `/knowledge` directories in a monorepo.** KEEP uses one zone at the root, not one per package. Splitting them creates artificial duplication.
- **Migrating pre-existing docs by hand.** Bypasses classification and elicitation. Use `/keep-init` (first time) or `/keep-observe ./folder/` (later) to produce the ingestion proposal, then `/keep-compile` to migrate with provenance.

## Repository layout reminder

```
/knowledge
├── docs/
│   ├── specs/             (per-domain subdirs; runbook/architecture are tags, not folders)
│   └── decisions/         (flat — sequential ADRs)
├── ideas/                 (inbox for half-formed proposals)
└── INDEX.md               (auto-generated from frontmatter)
```

That is the entire surface area. Nothing else needs to be added for KEEP to work.
