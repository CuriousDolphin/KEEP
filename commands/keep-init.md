---
description: Bootstrap KEEP in this repo — scaffold /knowledge and detect pre-existing docs
argument-hint: (no args)
---

Use the `keep` skill in **init mode** for this repository. Init is the only KEEP entry point that creates the `/knowledge` skeleton and bootstraps ingestion of pre-existing documentation; subsequent maintenance happens through `/keep-observe` and `/keep-compile`.

Follow this contract:

1. **Detect current state.** If `/knowledge` already exists, do **not** overwrite. Report what is there and confirm with the user whether to keep going (e.g. to re-scan for new pre-existing docs) or stop.

2. **Detect repo shape.** Look for monorepo signals — `pnpm-workspace.yaml`, `package.json` workspaces, Cargo workspace members, `go.work`, top-level `apps/` `packages/` `services/`. If found, ask the user to confirm the package list before scaffolding per-package subdirs under `specs/` and `runbooks/`. See `references/monorepo.md`.

3. **Scaffold the skeleton.** Create:
   - `/knowledge/docs/specs/`
   - `/knowledge/docs/decisions/`
   - `/knowledge/docs/architecture/`
   - `/knowledge/docs/runbooks/`
   - `/knowledge/tasks/`
   - `/knowledge/INDEX.md` (empty stub)
   Empty subdirectories are fine. Do NOT generate retroactive ADRs/specs for existing code.

4. **Scan for pre-existing docs.** Walk the repo (excluding `node_modules`, `.git`, `dist`, `build`, `target`, `vendor`, `.venv`) and collect candidate docs from common locations:
   - `README.md` (root and per-package)
   - `docs/`, `doc/`, `documentation/` (any depth)
   - `ARCHITECTURE.md`, `ARCHITECTURE/`, `DESIGN.md`, `DESIGN/`
   - `RUNBOOK.md`, `RUNBOOKS/`, `runbook/`, `runbooks/`
   - `notes/`, `design-notes/`, `rfc/`, `rfcs/`, `adr/`, `decisions/`
   - `wiki/`, `.wiki/`
   - Other top-level `*.md` that look structured (have headings, not just usage instructions)

5. **Classify each candidate.** Apply the heuristics in `references/brownfield.md` (heading patterns + keyword signals) to propose a target type (spec / ADR / architecture / runbook) and a target path. Flag mixed-content files as needing a user-driven split.

6. **Present the proposal — do not migrate yet.** Output the same classification format as `/keep-observe` in ingestion mode. Tell the user: *review the mapping, then run `/keep-compile` to migrate file-by-file with per-file approval and elicitation.*

7. **Wire the agent instructions.** If `CLAUDE.md`, `AGENTS.md`, or `.cursorrules` exists at repo root, append (do not overwrite) the short KEEP workflow snippet from `references/setup.md`. Otherwise, ask the user which file to create.

8. **Print a summary.** Skeleton created (or already present), monorepo packages detected, number of candidate docs by category, next step (`/keep-compile`).

**Hard rules**

- Never overwrite an existing `/knowledge` directory without explicit confirmation.
- Source doc folders are read-only — KEEP never modifies, moves, or deletes them.
- No migration during `/keep-init`. All ingestion writes happen through `/keep-compile` with explicit per-file approval.
- No retroactive backfill of code that has no pre-existing docs. KEEP grows from real changes.
