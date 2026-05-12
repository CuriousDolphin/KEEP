# Brownfield ingestion workflow — details

The summary is in `SKILL.md`. This file covers the operational detail: what `/observe` outputs in ingestion mode, what `/compile` does file-by-file, how to handle mixed-content source files, and the hard rules in full.

## When this workflow applies

Two scenarios:

1. **Adopting KEEP on an established project.** The team already has design docs, architecture notes, wiki exports, or markdown files scattered across the repo. They want to migrate this into the KEEP structure rather than start empty.

2. **Importing knowledge from other agents.** Earlier sessions with Claude or other coding assistants produced docs (typically in `./notes/`, `./docs/`, or scattered `*.md` files) that should now become durable knowledge.

The workflow is the same in both cases. The shared property is: there is pre-existing markdown content somewhere, and the goal is to fold it into `/knowledge/docs/` properly.

## Step 1 — `/observe ./folder/` classifies candidates

The user invokes `/observe` with a folder path instead of a diff. KEEP reads every file in the folder (recursively) and proposes a classification.

Example output:

```
Detected ingestion candidates (from ./old-docs/, 8 files):

Likely specs:
- ./old-docs/auth-design.md → knowledge/docs/specs/auth-service/jwt.md
  (describes JWT validation, issuer/audience, expiry — fits the spec template)

Likely architecture:
- ./old-docs/system-overview.md → knowledge/docs/architecture/overview.md
  (describes the high-level topology)

Likely runbooks:
- ./old-docs/deployment-runbook.md → knowledge/docs/runbooks/shared/deployment.md
  (already structured as symptoms / causes / mitigation — clean fit)

Likely ADRs (require elicitation):
- ./old-docs/why-ray-serve.md  ← mentions alternatives but no explicit rejection rationale
  → knowledge/docs/decisions/ADR-NNNN-ray-serve.md (next available number)

Unclear (need user input):
- ./old-docs/notes-from-meeting-march.md  ← mixed content: rationale, action items, scratch
  → could split into 1 ADR + several task notes, or be discarded

No action suggested for:
- ./old-docs/old-todo-list.md  ← appears ephemeral
- ./old-docs/random-snippets.md  ← unstructured code fragments, not knowledge
```

`/observe` in ingestion mode is **classification-only** — it never writes to `/knowledge/`.

## Step 2 — user reviews

The user accepts, corrects, or drops each candidate:

- *"`auth-design.md` should actually be split — the first half is a spec, the second half is rationale that belongs in an ADR."*
- *"Skip the meeting notes."*
- *"Migrate the runbook as-is, no changes needed."*

This review is the cheap correction point. Any errors in classification are caught here, before any files are written.

## Step 3 — `/compile` migrates file-by-file

For each accepted candidate, `/compile` does the following:

### 3a. Read the source file

Load the full content of the source markdown.

### 3b. Map content into the target template

Take the template from `references/file_formats.md` for the target type. Map source content into the right sections:

- Headings in the source that match template fields (e.g. `## Edge cases`) are mapped directly.
- Content that obviously belongs to a section (e.g. a bulleted list of failure modes under a different heading) is moved to the matching section.
- Content that doesn't fit anywhere is kept verbatim in a `## Migration notes` section at the bottom of the new file, for later cleanup. Never discard source content silently.

### 3c. Run the elicitation protocol for missing high-stakes fields

If the source provides rejected alternatives but not the rationale for rejection (e.g. *"Considered KServe, BentoML, custom FastAPI"* with no follow-up), batch-ask the user:

> *"Migrating `why-ray-serve.md` into ADR-NNNN. The source lists KServe, BentoML, and custom FastAPI as alternatives but doesn't explain why each was rejected. Quick batch:*
> *1. Why was KServe rejected? (CRD complexity / Python ergonomics / not enough distributed support / other)*
> *2. Why was BentoML rejected? (limited distributed support / pricing / Python ergonomics / other)*
> *3. Why was custom FastAPI rejected? (scaling concerns / maintenance burden / other)"*

If elicitation cannot fill the gap (user declines), follow the partial-answer rule from `elicitation.md` — omit the section and insert a `<!-- TODO(KEEP): ... -->` marker.

### 3d. Quote source content verbatim

If the source says *"Postgres was chosen for ACID guarantees and operational simplicity"*, the migrated file says exactly that. Do **not** rewrite to *"PostgreSQL was selected based on transactional integrity and reduced operational overhead"* — even if you think the rewrite is clearer. The user's wording is evidence; preserving it makes the lineage auditable and avoids the trust loss that comes from agents "improving" content unbidden.

The only edits allowed during migration:

- Restructuring (moving content from one section to another to fit the template).
- Removing genuinely redundant repetition.
- Fixing obvious typos.

Anything more substantial requires asking.

### 3e. Add a provenance comment

At the end of the migrated file, add a markdown comment:

```md
<!-- Migrated from ./old-docs/auth-design.md on 2026-05-12 -->
```

If a file was split across multiple migrated files (one spec + one ADR from the same source), each migrated file gets the same provenance marker. This makes the source-to-target mapping discoverable.

### 3f. Update INDEX.md incrementally

The new file gets indexed under its domain. Entities and flows are extracted from the migrated content.

### 3g. Do not touch the source folder

The source files in `./old-docs/` are read-only as far as KEEP is concerned. They are never moved, deleted, modified, or renamed. If the user wants to archive or clean up the source folder, that's a manual step after migration is complete.

## Handling mixed-content source files

A common case: a source file like `notes-from-meeting-march.md` contains a mix of:

- Decision rationale (→ ADR)
- Action items (→ tasks)
- Implementation notes (→ either spec content or scratch)
- General discussion (probably discard)

`/compile` should propose a split rather than guess:

> *"`notes-from-meeting-march.md` looks like mixed content. I can propose a split: the section on Ray Serve choice → ADR, the bullet list of action items → tasks, the rest → discard. Want me to propose specific mappings, or do you want to review the file together first?"*

When splitting:

- Each resulting file gets its own provenance comment, referencing the source and noting which section it came from: `<!-- Migrated from ./old-docs/notes-from-meeting-march.md (Ray Serve discussion section) on 2026-05-12 -->`.
- The source file is still left intact in `./old-docs/`.

## Hard rules

- **Per-file approval.** Never migrate the whole folder in one shot. Each file gets a yes/no/edit pass. Migration en masse hides classification errors.
- **No paraphrasing of source content.** The user's words are evidence. Preserve wording verbatim except for restructuring, dedup, and typos.
- **Elicitation still applies.** Missing alternatives, edge cases, or root causes are still high-stakes during migration. A thin ADR migrated from a source that didn't capture rationale is worse than no ADR.
- **Provenance comments always.** Every migrated file carries a comment pointing to its source (or sources, for split content). This is non-negotiable.
- **Source files are never modified.** KEEP reads them; the user owns cleanup.
- **`/observe` never writes during ingestion classification.** It only proposes the plan.
- **`/compile` writes one file at a time with explicit approval.** No batch migration.
