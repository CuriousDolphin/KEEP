---
description: Classify the impact of changes (git diff or existing docs) on the knowledge layer
argument-hint: [branch | PR | tag | commit-range | folder-path]
---

Use the `keep` skill in **observe mode**. Source: `$ARGUMENTS`.

Resolve the source in this order:

1. **Explicit argument** — branch (`feature/auth-refresh`), PR (`PR#142` or URL), tag (`v2.3.0`), commit range (`main..feature/x`), or a folder of existing documentation (`./old-docs/`).
2. **Working diff** — `git diff` against the branch's merge base (or `main` / `master` / the default branch).
3. **Last commit** — `git diff HEAD~1` as final fallback.

When a PR or branch is given, also pull the commit list with messages — commit messages frequently carry rationale the diff alone hides ("switched from KServe because of CRD complexity"). When a tag is given, compare against the previous tag.

**Always combine signals.** Even when the source is git, scan for **pre-existing docs** under the touched paths (e.g. if the diff hits `services/auth/*`, check `services/auth/README.md`, `services/auth/docs/`, `docs/auth*`). Surface them as ingestion candidates *alongside* the diff classification, not as a separate run.

**Behavior**

- Categorize changes into: **Feature** (new or modified behavior), **Architecture** (structural / topological), **Decision** (a choice with rejected alternatives worth recording), **Operational** (something a runbook should capture).
- For each category, list the affected knowledge files and the kind of update they likely need (be specific about *what*, not just *which*).
- Quote rationale-bearing commit messages verbatim — phrases like "because", "instead of", "we tried", "this breaks", "incident", "rejected" are gold for ADR alternatives and runbook causes.
- If a folder of docs is the source (or surfaces alongside git), classify each doc by target type using the heuristics in `references/brownfield.md` and propose a target path.
- If a `CHANGELOG.md` exists, scan its unreleased section as a secondary signal — but PR commit history wins when both are available.

**Hard rules**

- Read-only. Never write to `/knowledge`.
- Conservative: a pure refactor with no semantic impact → say so and suggest no updates.
- If the source is empty, ambiguous, or unavailable, ask the user to clarify rather than guess.
- Quote evidence (commits, doc fragments) rather than paraphrasing — preserve the user's words so the user can verify.
- For ingestion: classification only. Never migrate from `/keep-observe`. Migration happens in `/keep-compile`, per-file, with approval.

See `references/observe_examples.md` and `references/brownfield.md` for worked examples and the full heuristic catalog.
