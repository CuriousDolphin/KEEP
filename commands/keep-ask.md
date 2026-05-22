---
description: Ask a question about the system — KEEP synthesizes an answer from /knowledge/ with citations
argument-hint: <natural-language question>
---

Question: `$ARGUMENTS`.

## Contract

1. **Read `/knowledge/INDEX.md` first** — auto-generated from frontmatter, lets you filter by domain/tags/type without opening files.
2. **Select 1-5 candidate files** matching the question via `description`, `tags`, `domain`. Prefer precision over recall.
3. **Open the candidates and synthesize**. Quote specific passages — don't paraphrase content you'd otherwise cite.
4. **Cite every load-bearing claim** with the file's `id` (e.g. `[SPEC-auth-jwt]`, `[ADR-0014]`).
5. **If `/knowledge/` doesn't cover the question, say so explicitly.** Don't fall back to generic knowledge presented as repo truth — that's the antipattern this command exists to prevent.

Output: a direct answer followed by a short *Sources* footer with cited ids and paths.

## List mode (no synthesis)

When the user says *"just list paths"* / *"give me the relevant files, don't synthesize"*, or when you only need to locate files before opening them yourself, skip steps 3-4 and return:

```
For "$ARGUMENTS":
- [SPEC-auth-jwt]   knowledge/docs/specs/auth/jwt.md   — JWT validation, issuer/audience checks, refresh flow
- [ADR-0014]        knowledge/docs/decisions/ADR-0014-ray-serve.md  — Ray Serve adoption (supersedes ADR-0007)
- [SPEC-auth-jwt-rotation]  knowledge/docs/specs/auth/jwt-rotation.md  — runbook: secret rotation
```

Same selection logic, no synthesis.

## Hard rules

- Read-only. No file writes.
- Answer must derive from `/knowledge/`. If you cite the code, mark it as *"from the code, not from the knowledge layer"*.
- Follow `related:` links when relevant — a spec may point to an ADR with the actual rationale; include both.
- Respect `status`: `superseded` is historical, `deprecated` is informational-but-moving-away. Never present them as current state without flagging.
- If `INDEX.md` is missing/empty, say so and suggest `/keep-init` (first time) or `/keep-compile` (to start populating).
