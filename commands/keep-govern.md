---
description: Detect stale docs, duplication, contradictions in /knowledge (suggestions only)
argument-hint: (no args)
---

Use the `keep` skill in **govern mode**. This is the periodic hygiene pass — run it occasionally (weekly at most), not every cycle.

Follow this contract (full detail in the skill's `/keep-govern` section):

Scan `/knowledge/docs` for:

- Files untouched for >6 months in areas where the corresponding code has changed.
- Pairs of files whose titles or content overlap heavily.
- ADRs that contradict each other without an explicit supersedes link.
- Files >~300 lines that should be split.
- Files containing `<!-- TODO(KEEP): ... -->` markers — incomplete knowledge to enrich now that context may be fresher.
- `INDEX.md` with more than ~5 single-entry domains — domain granularity too fine, suggest collapsing.

Scan `/knowledge/tasks` for:

- `active` tasks older than 30 days.
- Archived tasks older than 90 days.
- Tasks whose `topic` doesn't match any domain in `INDEX.md`.

**Output**

Group suggestions by action — *archive*, *merge*, *summarize*, *split*, *enrich*, *collapse* — each with a one-line rationale. Wait for explicit per-suggestion approval before applying anything.

**Hard rules**

- Suggestions only. Never auto-delete, auto-merge, or auto-rewrite.
- Preserve historical rationale. To remove a file, move it to `/knowledge/docs/_archive/`, never delete outright.
- Be conservative on the staleness heuristic — a doc untouched isn't necessarily stale if the code is also untouched.
