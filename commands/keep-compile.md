---
description: One-shot knowledge update from a diff/source — classify (observe phase) then write files (compile phase) + regenerate INDEX. Use `--dry` to stop after classification.
argument-hint: [source] [--dry]
---

Use the `keep` skill in **compile mode**. `/keep-compile` is the single command for "I made changes, update knowledge." It runs two phases in sequence:

1. **Observe** — classify the diff/source, list suggested updates by `id`. No writes.
2. **Compile** — apply each suggested update, run elicitation where needed, regenerate `INDEX.md`.

Pass `--dry` to stop after phase 1 (useful for "show me what would change before I commit to it"). Default: both phases.

Source resolution order:

1. **Explicit argument** — branch (`feature/auth-refresh`), PR (`PR#142`), tag, commit range (`main..feature/x`), or a folder of pre-existing docs to ingest (`./old-docs/`).
2. **Working diff** — `git diff` against the branch's merge base.
3. **Last commit** — `git diff HEAD~1` as final fallback.

When a PR or branch is given, pull commit list with messages — rationale often hides in commit prose ("switched from KServe because of CRD complexity"). Even when the source is a git artifact, scan for pre-existing docs near the touched paths and surface them as ingestion candidates alongside the diff classification.

---

## Phase 1 — Observe

1. **Read `INDEX.md`** to see what's already in the knowledge layer and which `id`s exist.
2. **Categorize each change** into one of:
   - **Feature** — new/modified behavior → spec change
   - **Architecture** — topology/boundary change → architecture-tagged spec
   - **Decision** — choice with rejected alternatives → new ADR
   - **Operational** — failure mode learned → runbook-tagged spec
   - **Refactor** — no semantic change → no knowledge update
3. **For each non-refactor change, list affected knowledge files by `id`** and what kind of update they need. Be specific about *what*, not just *which*.
4. **Surface rationale-bearing commit messages verbatim.** Phrases like "because", "instead of", "we tried", "this breaks", "incident", "rejected" are gold for ADR alternatives and runbook causes — quote them directly, do not paraphrase.
5. **Refactor-only diff?** Say so explicitly, propose ZERO updates, do not regenerate INDEX. A pure rename or find-and-replace with no behavior change is execution detail, not knowledge — capturing it is the antipattern KEEP exists to prevent.

Output of phase 1:

```
Detected changes:

Feature:
- <one-line behavioral change>

Decision:
- <one-line — candidate for new ADR>

Suggested knowledge updates:
- [SPEC-auth-jwt] update edge cases section to reflect <change>
- [ADR-NNNN] create new ADR for <decision> (number resolved in phase 2)
```

Reference files by their `id` (from frontmatter), not by raw paths. The `id` is stable; paths can move.

If `--dry` was passed, stop here.

---

## Phase 2 — Compile (write the files)

For each suggested update from phase 1:

- **New file** — create with full YAML frontmatter (see `references/file_formats.md`). The `description` field must be a *search snippet*, not a chapter heading. The `related` field uses convention-based patterns (`code:internal/auth/*_test.go matching TestJWT*`), not hard-coded paths.
- **Updated file** — make the smallest possible diff. Preserve human-written rationale verbatim. Update the `related` field if cross-references changed.
- **New ADR** — `ls decisions/` first, pick the next free `ADR-NNNN`. Run **batch elicitation** (2-3 correlated questions in one turn) for rejected alternatives and consequences if the diff doesn't establish them. Quote any commit messages that already capture rationale instead of re-asking.
- **ADR supersession** — when a new ADR replaces an old one: new file gets `supersedes: [ADR-NNNN]`; old file's `status` becomes `superseded` and gets a `## Superseded by` section appended. Body of the old ADR is never edited.
- **Migrating a pre-existing doc (ingestion)** — quote source content verbatim (restructure / dedup / typo fixes only). Add `<!-- Migrated from <source> on YYYY-MM-DD -->` comment. **Per-file approval** — never migrate a folder in one shot.
- **Scaffolding a brand-new domain** — when phase 1 introduced a domain with no prior `specs/<domain>/` content, propose two companion specs, not one:
  1. The behavioral spec for the feature that triggered the new domain (e.g. `specs/billing/invoices.md`).
  2. An architecture-tagged stub sketching how the domain fits (`specs/billing/topology.md` with `tags: [billing, architecture]`).

  Topology can be a stub (paragraph + `<!-- TODO(KEEP) -->` markers for unknowns) — the point is to plant the file with a search-snippet description so `/keep-ask` can route topology questions correctly. Skip this only when the new domain is a one-file utility with no topology to describe; note the omission in final output.

- **Suggest anchor candidates** — when writing a NEW spec (or substantially extending an existing one), scan the diff for **concrete values bound to identifiers** and propose them as anchor entries:

  - integer/float/string literals assigned to a `const`/`var` (Go) or top-level assignment (Python/TS)
  - function signatures named in the diff
  - test function names

  For each candidate, write it to the spec's `anchors:` frontmatter and surface to the user inline: *"I added 3 anchor candidates. Want to keep all? (y/n/edit)"*. **Do not invent anchors** for values not in the diff. **Do not anchor on free-form prose** — only on identifiers the code actually contains. See `references/file_formats.md` for the anchor schema.

  Anchored facts are what `/keep-check-drift` enforces deterministically. Specs without anchors still work — anchoring is opt-in — but they sit outside the drift gate.

### Regenerate `INDEX.md` (mandatory last step)

```bash
python <skill-path>/scripts/build_index.py knowledge/ --strict
```

The script walks `/knowledge/docs/` and `/knowledge/ideas/`, parses YAML frontmatter, emits a deterministic table-based `INDEX.md`. **Never hand-edit `INDEX.md`** — if the layout is wrong, fix the script. `--strict` exits with code 1 if any file has invalid/missing frontmatter; surface those to the user as a `/keep-govern` backlog item.

---

## Hard rules

- Minimal diffs. If you're rewriting a paragraph, stop and reconsider.
- Frontmatter is mandatory on every file you create. Files without frontmatter are unverified artifacts and they will lie.
- Verify filesystem state before sequential or set-based claims (next ADR number, whether a domain exists, whether a file is present). Check, don't assume.
- Never invent rejected alternatives, edge cases, or root causes. If the diff and elicitation can't establish them, omit the section and insert `<!-- TODO(KEEP): ... -->` for `/keep-govern` to surface later.
- Never auto-promote idea content into specs/ADRs. Surface and ask.
- Per-file approval for ingestion. No batch migration of doc folders.
- After writing the knowledge files, **always run `build_index.py`**. INDEX.md must be derived from frontmatter.

## Summary output

```
Phase 1 (observe):
- Detected 3 changes: 2 features, 1 decision
- Suggested updates: 1 new ADR, 1 new spec, 1 spec update

Phase 2 (compile):
Created:
- [ADR-0015] knowledge/docs/decisions/ADR-0015-dual-secret-rotation.md
- [SPEC-auth-refresh] knowledge/docs/specs/auth/refresh.md

Updated:
- [SPEC-auth-jwt] edge cases (grace window now applies)

Regenerated:
- INDEX.md (12 entries: 7 specs, 4 ADRs, 1 idea)
```
