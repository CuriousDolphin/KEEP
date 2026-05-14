---
description: Bootstrap KEEP in this repo — scaffold /knowledge, set up frontmatter conventions, detect pre-existing docs
argument-hint: (no args)
---

Use the `keep` skill in **init mode**. Init is the only entry point that creates the `/knowledge` skeleton and bootstraps ingestion. Maintenance afterward happens through `/keep-observe` and `/keep-compile`.

Follow this contract:

1. **Detect current state.** If `/knowledge` already exists, do not overwrite. Report and confirm with the user.

2. **Detect repo shape.** Look for monorepo signals (`pnpm-workspace.yaml`, `package.json` workspaces, Cargo workspace, `go.work`, top-level `apps/` `packages/` `services/`). If found, ask the user to confirm the package list before scaffolding per-package subdirs.

3. **Scaffold the skeleton:**
   ```
   /knowledge/
   ├── docs/
   │   ├── specs/             (per-domain subdirs)
   │   └── decisions/         (flat, ADR-NNNN-slug.md)
   ├── ideas/                 (inbox for half-formed proposals)
   └── INDEX.md               (auto-generated stub)
   ```
   Empty subdirectories are fine. Do NOT generate retroactive ADRs/specs for existing code.

4. **Bundle the build_index script.** Verify `scripts/build_index.py` is reachable from the skill — `/keep-compile` calls it.

5. **Scan for pre-existing docs.** Walk the repo (excluding `node_modules`, `.git`, `dist`, `build`, `target`, `vendor`, `.venv`) and collect candidates from:
   - `README.md` at root and in any package
   - `docs/`, `doc/`, `documentation/` (any depth)
   - `ARCHITECTURE.md`, `ARCHITECTURE/`, `DESIGN.md`, `DESIGN/`
   - `RUNBOOK.md`, `RUNBOOKS/`, `runbook/`, `runbooks/`
   - `notes/`, `design-notes/`, `rfc/`, `rfcs/`, `adr/`, `decisions/`
   - `wiki/`, `.wiki/`

6. **Classify each candidate** using the heuristics in `references/brownfield.md`. Propose a target type (spec / adr / spec+runbook tag / spec+architecture tag) and target path. Flag mixed-content files as needing user-driven split.

7. **Present the ingestion proposal — do NOT migrate yet.** Tell the user: *review the mapping, then run `/keep-compile` to migrate file-by-file with per-file approval and frontmatter generation.*

8. **Wire the agent instructions.** Append to `CLAUDE.md` / `AGENTS.md` / `.cursorrules` (whichever exists at repo root) the KEEP workflow snippet from `references/setup.md`. The snippet uses **hard read-path triggers** — without those the agent will not consult `/knowledge` on questions.

9. **Print a summary.** Skeleton state, packages detected, candidate doc count by category, next step (`/keep-compile`).

**Hard rules**

- Never overwrite an existing `/knowledge` directory without explicit confirmation.
- Source doc folders are read-only — KEEP never modifies, moves, or deletes them.
- No migration during init. All ingestion writes happen through `/keep-compile`.
- No retroactive backfill of code that has no pre-existing docs. KEEP grows from real changes.
- Every file created by `/keep-init` (stub INDEX.md, example spec/ADR if any) must include valid YAML frontmatter from the start.
