---
description: List knowledge files relevant to a topic (low-level — for synthesis prefer /keep-ask)
argument-hint: <topic>
---

Use the `keep` skill in **retrieve mode**. Topic: `$ARGUMENTS`.

> **For most questions, prefer `/keep-ask <question>`.** It synthesizes an answer with citations. Use `/keep-retrieve` only when you need the *paths* to open files yourself — for example, when about to implement and you want the spec open in your editor.

Follow this contract (full detail in the skill's `/keep-retrieve` section):

- Read `INDEX.md` first. Filter by `domain`, `tags`, and `description` substring match.
- Return a short list grouped by `type` and tag (Specs / ADRs / Runbooks / Architecture). Each item: id, title, path. No content — just identification.
- Cap at ~10 entries. If the topic is too broad to filter, ask the user to narrow.

**Hard rules**

- Read-only. No file writes.
- No speculation. Stick to what is indexed in `INDEX.md`.
- If `INDEX.md` is missing or empty, say so and suggest `/keep-init` (first time) or `/keep-observe` + `/keep-compile` (to populate).
- This command is *low-level*. For "how does X work?" questions, the right command is `/keep-ask`, not `/keep-retrieve`.
