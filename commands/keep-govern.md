---
description: Periodic hygiene check — detect entropy in /knowledge over time (stale, duplicated, oversized). Suggestions only.
argument-hint: (no args)
---

Use the `keep` skill in **govern mode**. Run periodically (weekly at most), not every cycle.

> **Difference from `/keep-check-drift`.** Drift detection answers *"does this specific diff contradict knowledge now?"* and runs on every change. Govern answers *"what entropy has accumulated in knowledge over time?"* and runs occasionally. A file can pass drift (matches current code) but fail govern (stale, oversized, duplicated). Both are detectors — neither modifies files.

Follow this contract (full detail in the skill's `/keep-govern` section):

**Scan `/knowledge/docs` and `/knowledge/ideas` for:**

- Files with `status: accepted` not touched in >6 months where the corresponding code area has changed (use the `related:` patterns to map).
- Pairs of files whose `description` or content overlap heavily — duplication candidates.
- ADRs that contradict each other without an explicit `supersedes`/`refines` link.
- Files >~300 lines that should be split.
- Files containing `<!-- TODO(KEEP): ... -->` markers — incomplete knowledge to enrich now that context may be fresher.
- Ideas in `status: draft` older than 30 days — either promote them or mark them `deprecated`.
- Specs without `related:` entries pointing to code or tests — the link is what makes drift detection possible.
- INDEX.md regeneration sanity: run `scripts/build_index.py --dry-run` and compare against the on-disk INDEX. If they differ, INDEX was hand-edited and needs regeneration.
- Stray directories that shouldn't be there: `/knowledge/tasks/`, `/knowledge/architecture/`, `/knowledge/runbooks/` — these were v1 layout. Suggest folding their contents into `/knowledge/docs/specs/` with appropriate tags, or removing them if they were never used.

**Output**

Group suggestions by action — *archive*, *merge*, *summarize*, *split*, *enrich*, *collapse*, *promote*, *deprecate* — each with a one-line rationale. Wait for explicit per-suggestion approval before applying anything.

**Hard rules**

- Suggestions only. Never auto-delete, auto-merge, or auto-rewrite.
- Preserve historical rationale. To remove a file, move it to `/knowledge/docs/_archive/`, never delete outright.
- Be conservative on the staleness heuristic — a doc untouched isn't necessarily stale if the code is also untouched.
- Govern never blocks anything. If you want enforcement on a specific diff, use `/keep-check-drift`.
