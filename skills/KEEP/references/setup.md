# Setup — adoption guide

This file covers first-time adoption: the manual equivalents of `scripts/init.sh`, where the canonical AGENTS.md snippet lives, the common mistakes, and how to verify the loop works on a real change.

## Step 1 — scaffold the layout

The recommended entry point is **`bash <skill-path>/scripts/init.sh`** — run from your repo root. It scaffolds the directory layout, detects monorepo shape, scans for pre-existing docs, and appends the KEEP workflow snippet to whichever instruction file exists at the repo root (`AGENTS.md` / `CLAUDE.md` / `.cursorrules`). It refuses to overwrite an existing `/knowledge/`.

The manual equivalent, from the repo root:

```
mkdir -p knowledge/docs/specs knowledge/docs/decisions knowledge/ideas
touch knowledge/INDEX.md
```

Empty subdirectories are fine. Do **not** generate retroactive specs, ADRs, or architecture docs for the existing code. The first real updates come from the next meaningful change via `/keep-compile`.

### Monorepo

`init.sh` auto-detects monorepo shape (`pnpm-workspace.yaml`, `go.work`, `Cargo.toml`, top-level `apps/` `packages/` `services/`). Per-package subdirectories under `specs/` are created lazily — on the first `/keep-compile` run that touches each package. You don't have to pre-create them.

### Pre-existing docs

`init.sh` scans the canonical doc locations (`README.md`, `docs/`, `ARCHITECTURE.md`, `RUNBOOK.md`, `notes/`, `rfc/`, `adr/`, `decisions/`, `wiki/`, `*.md` at root) and lists candidates. The list is informational — nothing is migrated. To start migrating, run `/keep-compile ./docs/` (or whichever folder), which proposes a target type and path per file, then asks for per-file approval. See `references/brownfield.md` for the classification heuristics.

## Step 2 — tell agents how to use KEEP

`init.sh` appends the canonical snippet to whichever instruction file already exists at the repo root.

**The single source of truth for the snippet is the `## AGENTS.md snippet — install in the repo` section of `SKILL.md`.** Do not maintain a second copy here — the snippet drifts the moment two sources exist.

The snippet is intentionally hard:

- It frames `/keep-ask` as **mandatory** consultation before answering questions about behavior, design, history, or any uncertainty about conventions. The wording "Before answering ANY of these, run `/keep-ask <topic>` first" is the hammer that defeats the dominant failure mode (under-triggering on the read path).
- It instructs the agent to **say so explicitly** when `/keep-ask` returns no indexed knowledge, rather than falling back to generic knowledge presented as repo truth.
- It marks code changes that touch behavior/architecture/operations as **incomplete** until `/keep-compile` has run. That last sentence is the one that does most of the work — without it, knowledge updates get skipped under deadline pressure.
- It wires `/keep-check-drift` into the merge path (`exit-code 1 blocks merge`), and `/keep-govern` to weekly hygiene.
- It carves out `/keep-idea` for half-formed thoughts so they don't get lost in chat.

When auditing an adoption, the failure mode to look for is a softened snippet: someone removed the "mandatory" wording, or replaced "ANY of these" with "some of these", and the agent quietly stopped consulting `/knowledge` on questions. Restore the canonical version from `SKILL.md` and the read path comes back to life.

## Step 3 — optional starting content

If the team already has a few obvious decisions worth capturing (the kind of thing that comes up in every onboarding conversation — *"why are we on Ray Serve?"*, *"why do we use Auth0?"*), write one or two ADRs by hand on day one. This gives `/keep-ask` something substantive to return on the first question and sets the tone for the format.

Don't try to write more than three or four. The point is to seed, not to backfill.

## Step 4 — verify the loop works

On the next real code change, run the full cycle:

```
/keep-ask <topic>           # synthesize prior context, OR ask for list-only paths
[make the change]
/keep-compile               # classify + write + regen INDEX in one shot
                            # add --dry to preview without writing
```

If `/keep-compile` produces a sensible classification on real work and asks the right kind of question for ADRs/new specs, KEEP is set up correctly. If it produces nothing or produces too much, recalibrate — the command contract is narrow on purpose.

## Common adoption mistakes

- **Backfilling.** Trying to document the existing codebase before using KEEP. Don't. Document as code changes — or use `scripts/init.sh` + `/keep-compile ./old-docs/` if you genuinely have existing docs to migrate.
- **Treating KEEP as a task tracker.** KEEP only stores durable knowledge (specs, ADRs, ideas). Active work-in-progress belongs in your ticket tracker or agent task list. If a working note turns out to contain durable signal (a decision, a runbook step, a confirmed edge case), capture it via `/keep-compile`, not by dumping the note into `/knowledge`.
- **Running `/keep-govern` every cycle.** It is meant to be periodic — weekly at most. Running it constantly creates noise.
- **Multiple `/knowledge` directories in a monorepo.** KEEP uses one zone at the root, not one per package. Splitting them creates artificial duplication.
- **Migrating pre-existing docs by hand.** Bypasses classification and elicitation. Use `/keep-compile ./folder/` to produce the ingestion proposal with provenance.
- **Hand-editing `INDEX.md`.** It is regenerated by `scripts/build_index.py` on every `/keep-compile`. If the layout is wrong, fix the script, not the file.

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
