---
description: Bootstrap KEEP in this repo — scaffold /knowledge, install SPEC-000-keep, append the KEEP snippet to AGENTS.md/CLAUDE.md. Run once per repo.
argument-hint: (no args)
---

Use the `keep` skill in **init mode**. Call `/keep-init` when:

- The repo has no `/knowledge/` yet, or
- `/keep` (the dashboard) reported "uninitialized" and the user agreed to proceed, or
- The user explicitly says *"set up KEEP here"*, *"initialize KEEP"*, *"bootstrap the knowledge layer"*.

This command is a write operation. Never invoke it silently — always confirm with the user before running, even if they seem to want it. The output of `init.sh` modifies the repo; that's worth one "OK?".

Follow this contract:

1. **Resolve the skill path** from the location of this file — go up two directories from `commands/keep-init.md`.

2. **Confirm with the user** in one short sentence:

   > I'm about to scaffold `/knowledge/`, write a `SPEC-000-keep.md` describing KEEP's conventions, and append the KEEP workflow snippet to `AGENTS.md` (or `CLAUDE.md` / `.cursorrules` if either exists). Source files are never modified. OK to proceed?

   If the user says no, stop. Don't re-ask. Don't pre-empt with "just to confirm…".

3. **Run the bootstrap script** from the repo root:

   ```bash
   bash <skill-path>/scripts/init.sh
   ```

   Show the script's output verbatim. The script:
   - Creates `/knowledge/docs/specs/`, `/knowledge/docs/decisions/`, `/knowledge/ideas/`
   - Detects monorepo layout
   - Appends the KEEP snippet to whichever AI entry file exists (or creates `AGENTS.md` if none does)
   - Refuses to overwrite an existing `/knowledge/`

4. **Install `SPEC-000-keep.md`** into `/knowledge/docs/specs/`. The template lives at `<skill-path>/references/templates/SPEC-000-keep.md` — copy it verbatim. Adjust only the `created:` date in the frontmatter to today. The point of this file: KEEP's own conventions become a self-spec that survives even if the skill is later uninstalled, and the index lists KEEP itself as a domain.

5. **Run `build_index.py`** so the new spec appears in `INDEX.md`:

   ```bash
   python3 <skill-path>/scripts/build_index.py knowledge/
   ```

6. **Report state** — one paragraph max:

   ```
   KEEP initialized.

   Created:
     /knowledge/{docs/{specs,decisions},ideas,INDEX.md}
     /knowledge/docs/specs/keep/SPEC-000-keep.md
   Appended KEEP snippet to: AGENTS.md

   Suggested next step: /keep-compile ./docs/  (cordon-off pre-existing docs)
                       or /keep-compile         (compile from current diff)
   ```

7. **Mention CI/pre-commit but do not auto-install.** Append one sentence:

   > KEEP works on its own, but `/keep-check-drift` becomes a real enforcement gate when wired into CI or a pre-commit hook. Examples in `<skill-path>/references/setup.md` — run `/keep-ask "how do I wire drift into CI"` later if you want to set that up.

   Then stop. Do not write `.git/hooks/pre-commit` or `.github/workflows/keep.yml` yourself. Auto-installing into the user's git/CI configuration without explicit consent is the kind of "helpful surprise" that makes tools annoying.

**Hard rules**

- Always confirm before running `init.sh`. The user must explicitly OK the scaffold step.
- Never overwrite an existing `/knowledge/`. `init.sh` refuses this and so do you.
- Never touch `.git/`, `.github/`, or any CI configuration without an explicit request.
- After the command completes, do NOT also try to ingest pre-existing docs. `/keep-compile ./docs/` is its own opt-in step.
- Idempotency: re-running `/keep-init` on an already-initialized repo is a no-op + status print, not a re-scaffold.
