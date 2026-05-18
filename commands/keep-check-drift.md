---
description: Detect drift between code and /knowledge/ — deterministic anchor verification, CI-friendly (exit 1 on drift).
argument-hint: [--spec SPEC-id] [--changed] [--verbose]
---

Use the `keep` skill in **check-drift mode**. This is the **enforcement gate** — every PR, every pre-merge.

Unlike `/keep-govern` (hygiene over time, non-blocking), `/keep-check-drift` answers a precise question:

> *"Do the anchors declared in spec frontmatter still hold against the current code?"*

It's backed by `scripts/check_drift.py` — a deterministic, stdlib-only Python script. No LLM in the loop, no probabilistic verdicts. Exit code 1 blocks merge.

## How it works

For every anchor declared in spec frontmatter under `/knowledge/`, the script:

1. Resolves the binding (file + symbol + expected value/signature).
2. Reads the target file (Go, Python, or TypeScript).
3. Applies a language-specific regex matcher per anchor kind:
   - **const**: `const X = …` declaration matches expected value (whitespace-tolerant)
   - **function**: signature equality (params + return)
   - **test**: test exists AND is not skipped (`t.Skip`, `@pytest.mark.skip`, `test.skip(...)`, `xtest`, etc. all count as skipped)
   - **manual**: reported but never causes failure (external verification)
4. Aggregates results, prints a report, exits 0 (no drift) or 1 (drift).

## Invocation

```bash
python3 <skill-path>/scripts/check_drift.py [flags]
```

The agent, when asked to "check for drift", runs this script with appropriate flags rather than re-implementing the matching with an LLM. Flags:

- `--knowledge <path>` — defaults to `./knowledge`
- `--repo <path>` — defaults to the parent of `--knowledge`
- `--spec SPEC-id` — limit to a single spec
- `--changed` — only check anchors whose target file appears in `git diff` (CI / pre-commit shortcut)
- `--verbose` / `-v` — show OK results too, not just drift

Exit codes:
- `0` — no drift
- `1` — at least one anchor in drift or referencing a missing file
- `2` — usage error (e.g. `/knowledge/` missing)

## When to wire as a hook

The script is designed to be wired as a **git pre-commit hook** or a **CI step**:

```yaml
# Example .github/workflows/keep-drift.yml
- run: python3 skills/KEEP/scripts/check_drift.py --changed
```

Or as a local pre-commit hook (`.git/hooks/pre-commit`):

```bash
#!/bin/sh
python3 skills/KEEP/scripts/check_drift.py --changed
```

If exit code is 1, the PR is blocked until either the spec is updated or the code change is reverted.

## What this is NOT

- **Not a fixer.** The detector does not know whether the spec or the code is wrong. The user decides.
- **Not opinionated about prose.** Only `anchors:` frontmatter entries are checked. Plain-text claims in the spec body are not parsed.
- **Not an LLM call.** Pure regex. Predictable, fast (<30ms for hundreds of anchors), reproducible across runs.

## Hard rules

- No file writes. Detection only.
- Specs without `anchors:` in frontmatter contribute nothing to the check. They produce neither warnings nor failures — anchoring is opt-in per spec.
- A purely mechanical refactor (rename, find-and-replace) that does not change any anchored value or signature produces zero findings.
- If the script reports `missing` for an anchor (target file gone), that *is* drift — the symbol the spec points to no longer exists. Either restore the file or remove the anchor.

## Difference from `/keep-govern`

- `/keep-check-drift` — correctness **now**, on a specific diff, deterministic, blocks merge.
- `/keep-govern` — hygiene **over time**, on the whole knowledge base, suggestion-only, non-blocking.

A file can pass drift (matches today's code) but fail govern (stale, oversized, duplicated). And vice versa.
