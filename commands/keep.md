---
description: KEEP root command — shows /knowledge status as a dashboard, or bootstraps if missing. Run with no args.
argument-hint: (no args)
---

Use the `keep` skill in **status / bootstrap mode**. This is the entry point — what users (and you) run when they want to "see what's going on with KEEP in this repo" or when they don't remember which sub-command to call.

Follow this contract:

1. **Run the status script first**, from the repo root:
   ```bash
   python3 <skill-path>/scripts/status.py
   ```
   Use the skill's installed path (you can find it from the path of THIS file — go up two levels from `commands/keep.md`).

2. **Branch on the exit code:**

   - **Exit 0** — `/knowledge/` exists and the status was printed. Show the user the script's output verbatim. Then add a one-line reminder of the five command verbs (see below). Stop.

   - **Exit 2** — `/knowledge/` does not exist. KEEP is not initialized in this repo. Tell the user:

     > KEEP is not initialized in this repo. Run `/keep-init` to scaffold `/knowledge/`, install `SPEC-000-keep.md`, and append the KEEP workflow snippet to `AGENTS.md` (or `CLAUDE.md` / `.cursorrules`). It will ask for explicit confirmation before writing anything.

     Do NOT silently run `/keep-init` yourself — `/keep` is read-only by contract. The user invokes init explicitly.

     If the user seems unsure whether KEEP is right for the repo, give the two-sentence pitch (*a living knowledge layer for the repo — specs, ADRs, ideas, with anchored facts that drift-check catches deterministically*) and let them decide.

3. **After either path, append this one-line command map:**

   ```
   /keep-ask <question>   → read /knowledge with citations
   /keep-compile [--dry]  → classify diff/source, write/update specs/ADRs, regen INDEX
   /keep-check-drift      → verify anchors against current code (CI-friendly, exit 1 on drift)
   /keep-govern           → periodic hygiene (run weekly)
   /keep-idea <thought>   → capture a parked idea durably
   ```

**Hard rules**

- Never modify `/knowledge/` from this command. It is read-only — status or bootstrap only.
- Do not attempt to bootstrap silently. The user must explicitly approve `init.sh` on exit-2.
- Resolve the skill path from the location of `commands/keep.md`. Hardcoding paths breaks portability.
- If `status.py` errors with anything other than exit 2 (e.g. permissions), surface the error verbatim and stop — don't try to "fix" it.
