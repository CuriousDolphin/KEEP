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

`/keep-init` appends the snippet below to whichever instruction file already exists at the repo root (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`). If none exists, it asks which to create. The wording is intentionally short:

```md
## KEEP workflow

This repository uses KEEP (Knowledge Engine for Engineering Persistence).
Durable knowledge lives in `/knowledge/docs`. Ephemeral task notes live in `/knowledge/tasks`.

Workflow:
- First-time setup: `/keep-init` scaffolds `/knowledge` and proposes how to migrate
  any existing docs. Migration happens through `/keep-compile`, one file at a time.
- Before working on a topic: `/keep-retrieve <topic>` to load focused context.
- After non-trivial code changes: `/keep-observe` (accepts the current diff, or a
  branch/PR/tag, or `./folder/` for ingesting existing docs). It also opportunistically
  scans for pre-existing docs near the touched paths.
- Then `/keep-compile` to apply the suggested knowledge updates.
- Periodically: `/keep-govern` to detect stale or duplicated knowledge.

For ADRs and new specs, `/keep-compile` runs an elicitation interview (in batch) before
writing — expect a small number of targeted questions about rejected alternatives, edge
cases, or root causes when the diff alone doesn't establish them.

Task files (in `/knowledge/tasks/`) carry a minimal YAML frontmatter (`status`, `created`,
`topic`) which is managed automatically by `/keep-compile`. You don't need to maintain it
by hand.

The knowledge layer is not optional — code changes that affect behavior, architecture,
or operations should be reflected in `/knowledge` before the work is considered complete.
```

That last sentence is the one that does most of the work. Without it, knowledge updates get skipped under deadline pressure.

## Step 3 — optional starting content

If the team already has a few obvious decisions worth capturing (the kind of thing that comes up in every onboarding conversation — "why are we on Ray Serve?", "why do we use Auth0?"), write one or two ADRs by hand on day one. This gives `/keep-retrieve` something to return immediately and sets the tone for the format.

Do **not** try to write more than three or four. The point is to seed, not to backfill.

## Step 4 — verify the loop works

On the next real code change, run the full cycle:

```
/keep-retrieve <topic>      # before starting
[make the change]
/keep-observe               # classify impact (+ nearby docs)
/keep-compile               # apply updates (with batch elicitation if needed)
```

If `/keep-compile` produces a sensible diff on real work and asks the right kind of question for ADRs/new specs, KEEP is set up correctly. If it produces nothing or produces too much, recalibrate: the command contract is narrow on purpose.

## Common adoption mistakes

- **Backfilling.** Trying to document the existing codebase before using KEEP. Don't. Document as code changes — or use `/keep-init` and brownfield ingestion if you genuinely have existing docs to migrate.
- **Skipping `/keep-observe`.** Going straight to `/keep-compile` produces lower-quality updates because there is no classification step to anchor the changes.
- **Letting tasks leak into docs.** Tasks are ephemeral. If something belongs in durable memory, it goes through `/keep-compile`'s promotion mechanism into `/knowledge/docs`, not by copying task content directly.
- **Running `/keep-govern` every cycle.** It is meant to be periodic — weekly at most. Running it constantly creates noise.
- **Multiple `/knowledge` directories in a monorepo.** KEEP uses one zone at the root, not one per package. Splitting them creates artificial duplication.
- **Migrating pre-existing docs by hand.** Bypasses classification and elicitation. Use `/keep-init` (first time) or `/keep-observe ./folder/` (later) to produce the ingestion proposal, then `/keep-compile` to migrate with provenance.

## Repository layout reminder

```
/knowledge
├── docs/
│   ├── specs/             (per-package subdirs in monorepo)
│   ├── decisions/         (flat — sequential ADRs)
│   ├── architecture/      (flat — one file per area)
│   └── runbooks/          (per-package subdirs in monorepo)
├── tasks/                 (with auto-managed frontmatter)
└── INDEX.md
```

That is the entire surface area. Nothing else needs to be added for KEEP to work.
