# Setup — adoption guide

This file covers first-time adoption: the canonical AGENTS.md snippet, common mistakes, how to verify the loop works on a real change, and how to wire `/keep-check-drift` into CI or a pre-commit hook for full enforcement.

## Step 1 — initialize KEEP in the repo

The recommended entry point is **`/keep-init`** — a slash command that:

1. Asks for explicit consent (one prompt).
2. Runs `scripts/init.sh` to scaffold `/knowledge/docs/{specs,decisions}/`, `/knowledge/ideas/`, and `INDEX.md`. Detects monorepo shape. Appends the KEEP workflow snippet to `AGENTS.md` (or `CLAUDE.md` / `.cursorrules` if either exists).
3. Installs `SPEC-000-keep.md` from `references/templates/` into `/knowledge/docs/specs/keep/`. This self-spec documents KEEP's conventions inside the knowledge layer, so the conventions survive the skill being uninstalled.
4. Runs `scripts/build_index.py` so the new spec appears in `INDEX.md`.

The manual equivalent for users who can't run slash commands:

```bash
bash <skill-path>/scripts/init.sh
cp <skill-path>/references/templates/SPEC-000-keep.md knowledge/docs/specs/keep/
# Then edit the created: date in the SPEC frontmatter, and run:
python3 <skill-path>/scripts/build_index.py knowledge/
```

`init.sh` refuses to overwrite an existing `/knowledge/`. Re-running `/keep-init` on an already-initialized repo is a no-op + status print.

Empty subdirectories are fine. Do **not** generate retroactive specs, ADRs, or architecture docs for the existing code. The first real updates come from the next meaningful change via `/keep-compile`.

### Monorepo

`init.sh` auto-detects monorepo shape (`pnpm-workspace.yaml`, `go.work`, `Cargo.toml`, top-level `apps/` `packages/` `services/`). Per-package subdirectories under `specs/` are created lazily — on the first `/keep-compile` run that touches each package. You don't have to pre-create them.

### Pre-existing docs — cordon-off by default

If the repo has pre-existing docs (`docs/`, `ARCHITECTURE.md`, `README.md`, `wiki/`, etc.), KEEP's default treatment is to **cordon them off, not migrate them**. The reasoning is that a stale spec is worse than a missing spec — it poisons `/keep-ask` with confidently-wrong claims.

After `/keep-init`, run:

```
/keep-compile ./docs/
```

This writes a single ADR (`ADR-NNNN-legacy-docs-cordoned.md`) that declares the folder out of scope for KEEP. The files stay where they are, untouched. `/keep-ask` won't load them; `/keep-check-drift` won't verify against them.

Specific files can still be promoted into the knowledge layer one at a time with `--migrate`, after the user reads and verifies them:

```
/keep-compile docs/auth/jwt.md --migrate
```

The `--migrate` flag is the user signing off on currency. Batch migration of a folder is intentionally not supported. See `references/brownfield.md` for the full rationale and classification heuristics.

## Step 2 — tell agents how to use KEEP

`init.sh` appends the canonical KEEP snippet to whichever instruction file already exists at the repo root (`AGENTS.md` / `CLAUDE.md` / `.cursorrules`). If none exists, it creates `AGENTS.md`.

**The single source of truth for the snippet is the `## AGENTS.md snippet — install in the repo` section of `SKILL.md`.** Do not maintain a second copy here — the snippet drifts the moment two sources exist.

The snippet is intentionally hard:

- It frames `/keep-ask` as **mandatory** consultation before answering questions about behavior, design, history, or any uncertainty about conventions. The wording "Before answering ANY of these, run `/keep-ask <topic>` first" is the hammer that defeats the dominant failure mode (under-triggering on the read path).
- It instructs the agent to **say so explicitly** when `/keep-ask` returns no indexed knowledge, rather than falling back to generic knowledge presented as repo truth.
- It marks code changes that touch behavior/architecture/operations as **incomplete** until `/keep-compile` has run.
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

## Step 5 — wire drift into CI / pre-commit (recommended, not auto-installed)

`/keep-check-drift` is deterministic — `scripts/check_drift.py` exits 1 on any drifted anchor. That makes it a real enforcement gate when wired into your team's merge path. KEEP does NOT install hooks or CI workflows automatically (that kind of "helpful surprise" makes tools annoying); instead, here are two copy-paste templates.

### Pre-commit hook

Save as `.git/hooks/pre-commit` (or wire via [`pre-commit`](https://pre-commit.com/) if your team uses that framework):

```bash
#!/usr/bin/env bash
# KEEP — block commits that drift anchored knowledge.
# Only checks anchors whose target file appears in `git diff` (--changed flag).
set -e

SKILL_PATH="${SKILL_PATH:-$HOME/.claude/skills/keep}"   # adjust if installed elsewhere
if [ ! -f "$SKILL_PATH/scripts/check_drift.py" ]; then
    echo "warning: KEEP skill not found at $SKILL_PATH — skipping drift check"
    exit 0
fi

python3 "$SKILL_PATH/scripts/check_drift.py" --knowledge knowledge --changed
```

Make it executable: `chmod +x .git/hooks/pre-commit`. The hook is local — share it via documentation or a setup script in your repo, not via git itself.

### GitHub Action

Save as `.github/workflows/keep-check-drift.yml`:

```yaml
name: KEEP — check drift
on:
  pull_request:
    paths:
      - 'knowledge/**'
      - '**.go'
      - '**.py'
      - '**.ts'
      - '**.tsx'

jobs:
  check-drift:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.11' }
      - name: Clone KEEP skill
        run: git clone --depth 1 https://github.com/CuriousDolphin/KEEP /tmp/keep
      - name: Run drift check
        run: python3 /tmp/keep/skills/KEEP/scripts/check_drift.py --knowledge knowledge
```

Tweak the checkout step to use whatever distribution mechanism your team has standardized on (vendored copy, internal mirror, the official npm `skills.sh` installer in a previous step, etc.).

### Why not auto-install?

Two reasons. First, writing into `.git/hooks/` or `.github/workflows/` without explicit consent is invasive — the user might already have hooks they care about. Second, the right hook depends on the team's conventions (do they use `pre-commit` the framework? do they use GitHub Actions or GitLab CI or CircleCI? where is the KEEP skill checked out?). A copy-paste template the user adjusts is more honest than a guessed default.

## Common adoption mistakes

- **Backfilling.** Trying to document the existing codebase before using KEEP. Don't. Document as code changes — or cordon-off existing docs and migrate them piecemeal with `--migrate` if a specific file proves valuable.
- **Migrating an entire docs folder.** The default is cordon-off, not ingest. The friction of `--migrate` per file is the feature — it forces the user to read each file and confirm it's current.
- **Treating KEEP as a task tracker.** KEEP only stores durable knowledge (specs, ADRs, ideas). Active work-in-progress belongs in your ticket tracker or agent task list.
- **Running `/keep-govern` every cycle.** It is meant to be periodic — weekly at most. Running it constantly creates noise.
- **Multiple `/knowledge` directories in a monorepo.** KEEP uses one zone at the root, not one per package. Splitting them creates artificial duplication.
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
