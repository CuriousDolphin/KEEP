---
description: Apply suggested knowledge updates from /keep-observe — writes knowledge files and regenerates INDEX.md
argument-hint: (no args — uses last /keep-observe output)
---

Use the `keep` skill in **compile mode**. Take the previous `/keep-observe` (or `/keep-init`) output as input. If neither was run in this session, run `/keep-observe` first.

Follow this contract (full detail in the skill's `/keep-compile` section):

**For each suggested update**

- **New file** — create with full YAML frontmatter (see `references/file_formats.md`). The `description` field must be a *search snippet*, not a chapter heading. The `related` field must use convention-based patterns (`code:internal/auth/*_test.go matching TestJWT*`), not hard-coded paths.
- **Updated file** — make the smallest possible diff. Preserve human-written rationale verbatim. Update the `related` field if cross-references changed.
- **New ADR** — `ls` `decisions/` first, pick the next free `ADR-NNNN`. Run **batch elicitation** for rejected alternatives and consequences if the diff doesn't establish them. Cap at 3 questions per turn. Quote any commit messages that already capture rationale instead of re-asking. See `references/elicitation.md`.
- **ADR supersession** — when a new ADR replaces an old one, update both: new file gets `related: [adr:ADR-NNNN]` with "supersedes"; old file's `status` becomes `superseded` and gets a `## Superseded by` section at the bottom. Body of the old ADR is never edited.
- **Migrating a pre-existing doc (ingestion)** — quote source content verbatim (restructure / dedup / typo fixes only). Add `<!-- Migrated from <source> on YYYY-MM-DD -->` comment. Per-file approval — never migrate a folder in one shot.

**At the end of every /keep-compile run**

**Regenerate `INDEX.md` by running the bundled script**:

```bash
python <skill-path>/scripts/build_index.py knowledge/ --strict
```

The script walks `/knowledge/docs/` and `/knowledge/ideas/`, parses YAML frontmatter, and emits a deterministic table-based `INDEX.md`. **Never hand-edit `INDEX.md`** — if the layout is wrong, fix the script.

`--strict` exits with code 1 if any file has invalid/missing frontmatter. Surface those to the user.

**Hard rules**

- Minimal diffs. If you're rewriting a paragraph, stop and reconsider.
- Frontmatter is mandatory on every file you create. Files without frontmatter are unverified artifacts and they will lie.
- Verify filesystem state before sequential or set-based claims (next ADR number, whether a domain exists, whether a file is present). Check, don't assume.
- Never invent rejected alternatives, edge cases, or root causes. If the diff and elicitation can't establish them, omit the section and insert `<!-- TODO(KEEP): ... -->` for `/keep-govern` to surface later.
- Never auto-promote idea/task content into specs/ADRs. Surface and ask.
- Per-file approval for ingestion. No batch migration of doc folders.
- After writing the knowledge files, **always run `build_index.py`**. INDEX.md must be derived from frontmatter, not from your judgment.

**Summary output**

```
Created:
- [ADR-0015] knowledge/docs/decisions/ADR-0015-dual-secret-rotation.md
- [SPEC-auth-refresh] knowledge/docs/specs/auth/refresh.md

Updated:
- [SPEC-auth-jwt] edge cases (grace window now applies)

Regenerated:
- INDEX.md (12 entries: 7 specs, 4 ADRs, 1 idea)
```
