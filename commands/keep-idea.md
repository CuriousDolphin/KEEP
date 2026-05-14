---
description: Capture a half-formed idea durably in /knowledge/ideas/ — not yet a spec or ADR
argument-hint: <idea description, free-form>
---

Use the `keep` skill in **idea-capture mode**. Idea: `$ARGUMENTS`.

This command exists for the case the user describes a proposal but doesn't want to implement it now — and doesn't want to lose it in chat history. Ideas live in `/knowledge/ideas/` with `type: idea` frontmatter. They are the bidirectional bridge between exploratory thinking and durable knowledge: an idea can later be promoted into a spec or ADR via `/keep-compile`.

Follow this contract (full detail in the skill's `/keep-idea` section):

1. **Search for prior art first.** Run a quick scan of `/knowledge/` for prior ideas, runbooks, or ADRs that overlap with the user's idea. If something exists, surface it before capturing — the user often half-remembers prior thinking and is asking precisely because they don't trust their memory.
2. **Create a file at `/knowledge/ideas/<YYYY-MM-DD>-<slug>.md`** with frontmatter:
   - `id: IDEA-<YYYY-MM-DD>-<slug>`
   - `type: idea`
   - `status: draft`
   - `domain: <inferred>`
   - `created: <today>`
   - `tags: [...]`
   - `related:` — link to the prior art you found, if any
3. **Body**: capture *what the user actually said*, not your interpretation. Add a "Context" section if there's clear triggering reason. Add an "Open questions" section listing what would need to be resolved before promoting.
4. **Do NOT create a spec or ADR yet.** That's premature. Ideas are deliberately separate from durable artifacts because the act of committing to a spec/ADR implies a decision has been made.
5. **Confirm the capture briefly to the user** — path of the file, prior art found, suggested next step (e.g., "this could become an ADR once we settle on dual-secret vs JWKS").

**Hard rules**

- Ideas live in `/knowledge/ideas/` only. Execution state (active work in progress) belongs in your ticket tracker or agent task list — KEEP only stores durable knowledge.
- Never auto-promote an idea into a spec/ADR. Promotion happens in `/keep-compile` with explicit user approval.
- Preserve the user's wording. The act of paraphrasing destroys signal — they will read this later and need to recognize their own thought.
- An idea can reference a runbook, an ADR, another idea — use `related:` to keep the graph connected.

**Lifecycle**

- `status: draft` — initial capture.
- Promoted to a spec/ADR via `/keep-compile` — the idea stays in `/knowledge/ideas/` with `status: deprecated` and `related:` pointing to the resulting spec/ADR. Don't delete; the original thought is part of the lineage.
- Dropped without promotion — `status: deprecated` with a note explaining why.
- `/keep-govern` flags ideas older than 30 days still in `status: draft`.
