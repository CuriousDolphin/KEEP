---
description: Apply the suggested knowledge updates from /keep-observe (writes to /knowledge)
argument-hint: (no args — uses last /keep-observe output)
---

Use the `keep` skill in **compile mode**. Take the previous `/keep-observe` output from this session as input. If none exists, run observe first and ask the user to confirm the suggested updates before applying them.

Follow this contract (full detail in the skill's `/keep-compile` section):

- **Minimal diffs.** Preserve human-written rationale verbatim. Append or amend; do not rewrite paragraphs.
- **Elicitation before writing high-stakes fields.** Batch mode for new ADRs and new specs (rejected alternatives, consequences, edge cases, acceptance criteria are correlated — ask once, write once). Reactive mode for runbooks and incremental updates. Cap at 3 questions per turn. See `references/elicitation.md`.
- **ADR numbering.** `ls` `decisions/` and use the next available integer. Never pre-number from memory or conversation flow.
- **ADR supersession.** When the new ADR replaces an old one, update both files and cross-link. See `references/file_formats.md`.
- **Cross-references.** A spec implementing an ADR links to it; an ADR affecting an architecture component is linked from that doc.
- **Task lifecycle.** Promote durable content out of `/knowledge/tasks` only with user approval. Transition `status: active → done | abandoned` and archive per `references/tasks.md`.
- **INDEX.md.** Incremental update — touch only the sections affected. Preserve everything else byte-for-byte. (First-ever `/keep-compile` regenerates from scratch.)
- **Brownfield ingestion.** When the input includes ingestion candidates, migrate one file at a time with explicit user approval. Map content into the target template, quote source content verbatim (no paraphrasing — restructure, dedup, fix typos only), run elicitation for missing high-stakes fields, add a provenance comment `<!-- Migrated from <source> on <date> -->`. Source files are never modified.
- **Summary.** Print created / updated / skipped (with reason) and any task promotions or archivals.

**Hard rules**

- Verify filesystem state before claiming sequential or set-based facts (next ADR number, whether a domain exists in INDEX, whether a file is present). Check, don't assume.
- Never invent rejected alternatives, edge cases, or root causes. If the diff and elicitation can't establish them, omit the section and insert `<!-- TODO(KEEP): ... -->` for `/keep-govern` to catch later.
- Never auto-promote task content into durable docs. Surface and ask.
- Never delete existing rationale — mark it superseded instead.
- For ingestion: per-file approval. No en-masse migration.
