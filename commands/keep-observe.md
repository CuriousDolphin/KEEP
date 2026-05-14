---
description: Classify the impact of changes (git diff, PR, folder of docs) on the knowledge layer — proposes updates only
argument-hint: [branch | PR | tag | commit-range | folder-path]
---

Use the `keep` skill in **observe mode**. Source: `$ARGUMENTS`.

Resolve the source in this order:

1. **Explicit argument** — branch (`feature/auth-refresh`), PR (`PR#142` or URL), tag (`v2.3.0`), commit range (`main..feature/x`), or a folder of existing documentation to ingest (`./old-docs/`).
2. **Working diff** — `git diff` against the branch's merge base.
3. **Last commit** — `git diff HEAD~1` as final fallback.

When a PR or branch is given, also pull the commit list with messages — rationale often hides in commit prose ("switched from KServe because of CRD complexity"). When a tag is given, compare against the previous tag.

**Combined git + docs signal.** Even when the source is a git artifact, scan for pre-existing docs under or near the touched paths and surface them as ingestion candidates *alongside* the diff classification.

**Behavior**

1. **Read `INDEX.md`** to know what's already in the knowledge layer and what `id`s exist.
2. **Categorize changes** into: **Feature** (new/modified behavior — affects specs), **Architecture** (topology/boundary changes — affects architecture-tagged specs), **Decision** (a choice with rejected alternatives — candidate for new ADR), **Operational** (failure mode learned — affects runbook-tagged specs), **Refactor** (no semantic change — no knowledge update needed).
3. **For each category, list affected knowledge files by `id`** and the kind of update they likely need. Be specific about *what*, not just *which*.
4. **Surface rationale-bearing commit messages verbatim** — phrases like "because", "instead of", "we tried", "this breaks", "incident", "rejected" are gold for ADR alternatives and runbook causes.
5. **Output the classification only.** No file modifications.

**Hard rules**

- Never modify any file under `/knowledge`. Classification only.
- Be conservative: a pure refactor → say so, suggest no updates.
- Quote evidence (commits, doc fragments) rather than paraphrasing — the user verifies against the original.
- Folder ingestion (`/keep-observe ./old-docs/`) classifies each doc by likely target type using the heuristics in `references/brownfield.md`. Migration still happens in `/keep-compile`, per-file, with approval.

**Output shape**

```
Detected changes:

Feature:
- <one-line behavioral change>

Decision:
- <one-line — candidate for new ADR>

Suggested knowledge updates:
- [SPEC-auth-jwt] update edge cases section to reflect <change>
- [ADR-NNNN] create new ADR for <decision> (number resolved at /keep-compile)
```

Always reference files by their `id` (from the frontmatter), not raw paths. The id is stable; paths can move.
