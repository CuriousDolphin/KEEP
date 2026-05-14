---
description: Detect drift between code changes and /knowledge/ — linter-style enforcement
argument-hint: [branch | PR | commit-range — defaults to working tree diff]
---

Use the `keep` skill in **check-drift mode**. Input source: `$ARGUMENTS` (defaults to working tree diff against the merge base / main).

This is the **enforcement mechanism**: every commit, every PR, every pre-merge gate. Unlike `/keep-govern` which surfaces cumulative entropy, `/keep-check-drift` answers a precise question:

> *"Does this specific change introduce a contradiction with the knowledge layer as it stands right now?"*

Follow this contract (full detail in the skill's `/keep-check-drift` section):

1. **Resolve the source** (working tree diff, branch, PR, commit range). If ambiguous, ask.
2. **Identify affected knowledge files** via the `related:` patterns in frontmatter. A diff touching `internal/auth/jwt.go` matches every spec/ADR whose `related` field has `code:internal/auth/*` or `code:internal/auth/jwt.go`.
3. **For each affected file, check three drift types:**
   - **Behavioral drift** — the diff changes a value or behavior that the spec explicitly documents (timeout values, error messages, endpoint paths, status codes). Surface the contradiction with the exact spec passage.
   - **Decisional drift** — the diff implements an approach that an ADR explicitly rejected. Quote the rejection rationale.
   - **Operational drift** — the diff changes failure modes that a runbook-tagged spec documents (recovery steps, mitigation triggers).
4. **Output linter-style findings.** One block per finding: file:line of the change, affected file id, quoted passage from the affected file, one-line suggestion of direction. No file modifications.
5. **Exit code:** `0` if no drift, `1` if drift detected. Usable as a git pre-commit hook or CI gate.

**Hard rules**

- No file writes. Detection only.
- The detector is *not* a fixer — it does not know whether the spec or the code is wrong. The user decides.
- Quote evidence verbatim. A finding without a specific quoted passage from the affected file is noise.
- Be conservative: a purely structural refactor (no behavioral change) should produce zero findings even if it touches files referenced by specs.

**Difference from `/keep-govern`** — read both contracts in the SKILL.md to understand:

- `/keep-check-drift` = correctness *now*, on a specific diff, blocks merge.
- `/keep-govern` = hygiene *over time*, on the whole knowledge base, non-blocking.

A file can pass drift (matches today's code) but fail govern (stale, oversized, duplicated). And vice versa.
