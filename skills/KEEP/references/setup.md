# Setting up KEEP in an existing repository

KEEP is *brownfield-first*. Adoption is meant to take an hour, not a week. There is no upfront documentation requirement; knowledge grows from real changes from the moment KEEP is in place.

## Step 1 — create the skeleton

From the repo root:

```bash
mkdir -p knowledge/docs/specs
mkdir -p knowledge/docs/decisions
mkdir -p knowledge/docs/architecture
mkdir -p knowledge/docs/runbooks
mkdir -p knowledge/tasks
touch knowledge/INDEX.md
```

Empty subdirectories are fine. Do **not** generate retroactive specs, ADRs, or architecture docs for the existing code. The first real updates will come from the next meaningful change via `/observe` and `/compile`.

### Monorepo setup

KEEP uses a single `/knowledge/` directory at the repository root, even for monorepos. There are no per-package knowledge directories. Inside the single zone, `specs/` and `runbooks/` get a per-package subdirectory; `decisions/` and `architecture/` stay flat. See the **Monorepo support** section in the main SKILL.md for details.

For a monorepo, also create the package subdirectories you already know about:

```bash
mkdir -p knowledge/docs/specs/auth-service
mkdir -p knowledge/docs/specs/inference-api
mkdir -p knowledge/docs/specs/shared
mkdir -p knowledge/docs/runbooks/auth-service
mkdir -p knowledge/docs/runbooks/shared
```

Empty subdirectories are still fine — they just establish the layout convention.

### Ingesting pre-existing documentation

If the repo already has a folder of design notes or wiki exports (e.g. `./old-docs/`), do **not** copy them into `/knowledge/docs/` by hand. Use the brownfield ingestion workflow:

1. Set up the skeleton above.
2. Run `/observe ./old-docs/` to get a classification of candidates.
3. Run `/compile` to migrate them file-by-file with elicitation.

This produces structured, cross-linked knowledge files instead of a dump of legacy markdown.

## Step 2 — tell agents how to use KEEP

Add a short section to whatever instruction file the agent reads (`CLAUDE.md` for Claude Code, `AGENTS.md` for some setups, `.cursorrules` for Cursor). The wording does not need to be elaborate — a short, direct snippet works best:

```md
## KEEP workflow

This repository uses KEEP (Knowledge Engine for Engineering Persistence).
Durable knowledge lives in `/knowledge/docs`. Ephemeral task notes live in `/knowledge/tasks`.

Workflow:
- Before working on a topic: `/retrieve <topic>` to load focused context.
- After non-trivial code changes: `/observe` (accepts the current diff, or a branch/PR/tag,
  or `./folder/` for ingesting existing docs).
- Then `/compile` to apply the suggested knowledge updates.
- Periodically: `/govern` to detect stale or duplicated knowledge.

For ADRs and new specs, `/compile` runs an elicitation interview (in batch) before writing —
expect a small number of targeted questions about rejected alternatives, edge cases,
or root causes when the diff alone doesn't establish them.

Task files (in `/knowledge/tasks/`) carry a minimal YAML frontmatter
(`status`, `created`, `topic`) which is managed automatically by `/compile`.
You don't need to maintain it by hand.

The knowledge layer is not optional — code changes that affect behavior, architecture,
or operations should be reflected in `/knowledge` before the work is considered complete.
```

That last sentence is the one that does most of the work. Without it, knowledge updates get skipped under deadline pressure.

## Step 3 — optional starting content

If the team already has a few obvious decisions worth capturing (the kind of thing that comes up in every onboarding conversation — "why are we on Ray Serve?", "why do we use Auth0?"), write one or two ADRs by hand on day one. This gives `/retrieve` something to return immediately and sets the tone for the format.

Do **not** try to write more than three or four. The point is to seed, not to backfill.

## Step 4 — verify the loop works

On the next real code change, run the full cycle:

```
/retrieve <topic>      # before starting
[make the change]
/observe               # classify impact
/compile               # apply updates (with batch elicitation if needed)
```

If `/compile` produces a sensible diff on real work and asks the right kind of question for ADRs/new specs, KEEP is set up correctly. If it produces nothing or produces too much, recalibrate: the four-command contract is narrow on purpose.

## Common adoption mistakes

- **Backfilling.** Trying to document the existing codebase before using KEEP. Don't. Document as code changes — or use brownfield ingestion if you genuinely have existing docs to migrate.
- **Skipping `/observe`.** Going straight to `/compile` produces lower-quality updates because there is no classification step to anchor the changes.
- **Letting tasks leak into docs.** Tasks are ephemeral. If something belongs in durable memory, it goes through `/compile`'s promotion mechanism into `/knowledge/docs`, not by copying task content directly.
- **Running `/govern` every cycle.** It is meant to be periodic — weekly at most. Running it constantly creates noise.
- **Multiple `/knowledge` directories in a monorepo.** KEEP uses one zone at the root, not one per package. Splitting them creates artificial duplication.

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
