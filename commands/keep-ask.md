---
description: Ask a question about the system — KEEP synthesizes an answer from /knowledge/ with citations
argument-hint: <natural-language question>
---

Use the `keep` skill in **ask mode**. Question: `$ARGUMENTS`.

Follow this contract (full detail in the skill's `/keep-ask` section):

1. **Read `/knowledge/INDEX.md` first.** It is auto-generated from frontmatter and lets you filter by domain, tags, type without opening files.
2. **Select 1-5 candidate files** that match the question, using `description`, `tags`, and `domain` fields. Prefer precision over recall — irrelevant context degrades reasoning.
3. **Open the candidates and synthesize an answer.** Quote specific passages — do not paraphrase content you would otherwise cite.
4. **Cite every load-bearing claim** with the file's `id` (e.g. `[SPEC-auth-jwt]`, `[ADR-0014]`). The id appears in the INDEX and at the top of the file's frontmatter.
5. **If the knowledge layer does NOT cover the question, say so explicitly.** Do not fall back to generic knowledge presented as repo truth — that is the antipattern this command exists to prevent.

**Hard rules**

- Read-only. No file writes.
- The answer must be derived from `/knowledge/`. If you cite the code, mark it clearly as "from the code, not from the knowledge layer".
- Follow `related:` links in frontmatter when they bear on the question — a spec may point to an ADR with the actual rationale, and you should include both.
- Respect `status`: `superseded` entries are historical; `deprecated` are still informational but moving away. Never present them as the current state without flagging.
- If `INDEX.md` is missing or empty, say so and suggest `/keep-init` (first time) or `/keep-observe` + `/keep-compile` (to start populating).

**Output shape**

A direct answer to the question, followed by a short "Sources" footer listing the cited file ids and paths.
