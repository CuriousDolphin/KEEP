---
description: Load focused KEEP context for a topic (read-only)
argument-hint: <topic>
---

Use the `keep` skill in **retrieve mode**. Topic: `$ARGUMENTS`.

Follow this contract (full detail in the skill's `/keep-retrieve` section):

- Read `INDEX.md` first as the primary map. Fall back to filename and content search across `/knowledge/docs/` only if INDEX is empty or doesn't cover the topic.
- Surface only files that touch the topic. Group output by category — Architecture / Decisions / Specs / Runbooks — and list paths, not contents. The user opens what they need.
- Prefer precision over completeness. Token budget is a real cost; irrelevant context degrades reasoning.

**Hard rules**

- Read-only. No file writes.
- No speculation from the code about what *might* be relevant. Stick to what is indexed.
- If `INDEX.md` is empty or missing, say so and suggest `/keep-init` (first-time setup) or `/keep-observe` + `/keep-compile` (to start populating it).
