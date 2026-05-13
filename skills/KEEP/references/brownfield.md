# Brownfield ingestion workflow — details

The summary is in `SKILL.md` (sections **The five commands** and **Pre-existing documentation auto-detection**). This file covers the operational detail: where to look, how to classify, what `/keep-observe` outputs in ingestion mode, what `/keep-compile` does file-by-file, how to handle mixed-content source files, and the hard rules in full.

## When this workflow applies

Three scenarios — all share the same machinery:

1. **First-time KEEP adoption on an established project.** Triggered by `/keep-init`. The team already has design docs, architecture notes, wiki exports, or markdown files scattered across the repo. KEEP scaffolds `/knowledge/` and produces an ingestion proposal for everything it finds.

2. **Explicit folder ingestion at any time.** Triggered by `/keep-observe ./some-folder/`. The user points KEEP at a specific folder (wiki export, dump from another agent, legacy doc tree) and gets back a classification.

3. **Combined git + docs signal during normal observation.** Triggered by `/keep-observe` with a git source. KEEP scans for pre-existing docs *near the touched code paths* and folds candidates into the same proposal as the diff classification.

The classification machinery is identical in all three. The only difference is the entry point and the scope of the scan.

## Where to scan

The canonical locations where pre-existing docs tend to live. Walk these recursively, excluding `node_modules`, `.git`, `dist`, `build`, `target`, `vendor`, `.venv`, and any path already under `/knowledge/`:

- `README.md` at the repo root and inside any package
- `docs/`, `doc/`, `documentation/` at any depth
- `ARCHITECTURE.md`, `ARCHITECTURE/`, `DESIGN.md`, `DESIGN/`
- `RUNBOOK.md`, `RUNBOOKS/`, `runbook/`, `runbooks/`
- `notes/`, `design-notes/`, `rfc/`, `rfcs/`, `adr/`, `decisions/`
- `wiki/`, `.wiki/`
- Top-level `*.md` other than typical project boilerplate (`LICENSE`, `CONTRIBUTING`, `CODE_OF_CONDUCT`, `CHANGELOG`)

When triggered from `/keep-observe` with a git source, also scan **paths adjacent to the diff**:

- For each file the diff touches, look in the same directory and one level up.
- For each top-level service / package the diff touches, look in its conventional doc paths (e.g. `services/auth/docs/`, `packages/foo/README.md`).
- Anything that matches one of the canonical patterns above counts as a candidate.

The point of the "near the diff" scan is to surface latent docs the user may have forgotten about — they often carry rationale that the commits don't.

## Classification heuristics

Two signal types, applied jointly:

- **Heading-pattern signal** — structural. Read the file's outline (top-level + second-level headings). Compare to the templates in `references/file_formats.md`.
- **Keyword signal** — semantic. Scan the body for marker phrases.

When the two signals disagree, **prefer the heading signal** — structure is more reliable than wording. When neither signal is conclusive, flag as **unclear** and ask the user.

### Likely spec

Heading patterns: `## Endpoint`, `## API`, `## Behavior`, `## Behaviour`, `## Requirements`, `## Acceptance criteria`, `## Inputs`, `## Outputs`, `## Validation`, `## Edge cases`, `## Errors`, `## Responses`.

Keyword signals: "shall", "must return", "is expected to", "given … when … then", "rejects", "accepts", "validates", "returns 4xx", "returns 5xx", request/response examples.

Target path: `/knowledge/docs/specs/<package-or-area>/<slug>.md`. Infer `<package>` from the source path (`services/auth/docs/jwt.md` → `specs/auth/jwt.md`). In single-package repos, use `<area>` from the filename or top heading.

### Likely runbook

Heading patterns: `## Symptoms`, `## Detection`, `## Cause`, `## Causes`, `## Mitigation`, `## Resolution`, `## Rollback`, `## Postmortem`, `## Recovery`, `## Triggers`, `## Alerts`.

Keyword signals: "alert", "page", "paged", "on-call", "oncall", "incident", "post-mortem", "RTO", "RPO", "SLO breach", "rollback", "5xx spike", "p99", "OOM", "OOMKilled", "circuit breaker", "kill switch".

Target path: `/knowledge/docs/runbooks/<package-or-area>/<slug>.md`, or `runbooks/shared/<slug>.md` for cross-cutting (e.g. deployment, CI/CD, secrets rotation).

### Likely ADR / decision

Heading patterns: `## Context`, `## Decision`, `## Status`, `## Alternatives`, `## Alternatives considered`, `## Consequences`, `## Trade-offs`, `## Rationale`, `## Why X`. Filenames starting with `ADR-`, `adr-`, `decision-`, or `rfc-`.

Keyword signals: "because", "instead of", "we chose", "we picked", "we rejected", "we tried … but", "supersedes", "deprecates", "the trade-off is", "the cost is", "the upside is", "vs.", "compared to".

Target path: `/knowledge/docs/decisions/ADR-NNNN-<slug>.md` where `NNNN` is the next available number. **Compute `NNNN` at compile time** by listing the directory; `/keep-observe` and `/keep-init` propose `ADR-NNNN` as a placeholder.

### Likely architecture

Heading patterns: `## Topology`, `## Components`, `## Component diagram`, `## Flows`, `## Data flow`, `## Boundaries`, `## System overview`, `## Overview`, `## Dependencies`, `## Integration points`.

Keyword signals: "service mesh", "boundary", "owns", "depends on", "produces", "consumes", "downstream", "upstream", "fan-out", topology diagrams (mermaid / ASCII / linked images).

Target path: `/knowledge/docs/architecture/<area>.md` (one file per package, plus `overview.md` for system-wide topology).

### Mixed content

A file matches multiple categories with substantive content for each (e.g. meeting notes that include a decision rationale + a runbook entry + scratch). Don't guess — **propose a split** at observe time and let `/keep-compile` confirm with the user before writing.

Typical splits:

- *meeting notes* → 1 ADR + a few task notes + discard the rest
- *design doc that grew over time* → 1 spec + 1 ADR (the rationale section) + 1 architecture file (the topology section)
- *postmortem* → 1 runbook + 1 ADR (if the fix involved a structural choice)

### Discard candidates

Pure code snippets, dependency lists, build instructions, user-facing how-tos, todo lists, contributor onboarding pages. These are not durable engineering knowledge in KEEP's sense. Flag them and let the user decide — but don't propose a target.

### Unclear

When heading and keyword signals disagree without a clear winner, or when neither signal is strong enough to be confident, list the file under **Unclear** with a one-line description and let the user classify it manually.

## Step 1 — observe (or init) classifies candidates

Whether triggered by `/keep-init`, `/keep-observe ./folder/`, or the "near the diff" scan inside `/keep-observe` with a git source, the output format is the same:

```
Detected ingestion candidates (from ./old-docs/, 8 files):

Likely specs:
- ./old-docs/auth-design.md → knowledge/docs/specs/auth-service/jwt.md
  (matches the spec template — has API/Behavior/Edge cases headings)

Likely architecture:
- ./old-docs/system-overview.md → knowledge/docs/architecture/overview.md
  (describes the high-level topology — components + dependencies)

Likely runbooks:
- ./old-docs/deployment-runbook.md → knowledge/docs/runbooks/shared/deployment.md
  (already structured as symptoms / causes / mitigation — clean fit)

Likely ADRs (require elicitation at compile time):
- ./old-docs/why-ray-serve.md → knowledge/docs/decisions/ADR-NNNN-ray-serve.md
  (mentions alternatives but no explicit rejection rationale — will need batch interview)

Mixed content (propose split):
- ./old-docs/notes-from-meeting-march.md
  → 1 ADR (Ray Serve discussion section) + task notes (action items) + discard the rest

Unclear (need user input):
- ./old-docs/auth-2023.md
  (could be an outdated spec or an architecture sketch — please classify)

No action suggested for:
- ./old-docs/old-todo-list.md  (appears ephemeral)
- ./old-docs/random-snippets.md  (unstructured code fragments — not durable knowledge)
```

The observation step is **classification-only** — it never writes to `/knowledge/`.

## Step 2 — user reviews

The user accepts, corrects, or drops each candidate. Typical interventions:

- *"`auth-design.md` should actually be split — the first half is a spec, the second half is rationale that belongs in an ADR."*
- *"Skip the meeting notes."*
- *"Migrate the runbook as-is, no changes needed."*
- *"Move `auth-2023.md` to discard — it's stale."*

This review is the cheap correction point. Any errors in classification are caught here, before any files are written.

## Step 3 — `/keep-compile` migrates file-by-file

For each accepted candidate, `/keep-compile` does the following:

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

If a file was split across multiple migrated files (one spec + one ADR from the same source), each migrated file gets the same provenance marker, qualified by the section it came from:

```md
<!-- Migrated from ./old-docs/notes-from-meeting-march.md (Ray Serve discussion section) on 2026-05-12 -->
```

This makes the source-to-target mapping discoverable.

### 3f. Update INDEX.md incrementally

The new file gets indexed under its domain. Entities and flows are extracted from the migrated content.

### 3g. Do not touch the source folder

The source files in `./old-docs/` (or wherever they live) are read-only as far as KEEP is concerned. They are never moved, deleted, modified, or renamed. If the user wants to archive or clean up the source folder, that's a manual step after migration is complete.

## Handling mixed-content source files

A common case: a source file like `notes-from-meeting-march.md` contains a mix of:

- Decision rationale (→ ADR)
- Action items (→ tasks)
- Implementation notes (→ either spec content or scratch)
- General discussion (probably discard)

`/keep-compile` should propose a split rather than guess:

> *"`notes-from-meeting-march.md` looks like mixed content. I can propose a split: the section on Ray Serve choice → ADR, the bullet list of action items → tasks, the rest → discard. Want me to propose specific mappings, or do you want to review the file together first?"*

When splitting:

- Each resulting file gets its own provenance comment, referencing the source and noting which section it came from.
- The source file is still left intact.

## Hard rules

- **Per-file approval.** Never migrate the whole folder in one shot. Each file gets a yes/no/edit pass. Migration en masse hides classification errors.
- **No paraphrasing of source content.** The user's words are evidence. Preserve wording verbatim except for restructuring, dedup, and typos.
- **Elicitation still applies.** Missing alternatives, edge cases, or root causes are still high-stakes during migration. A thin ADR migrated from a source that didn't capture rationale is worse than no ADR.
- **Provenance comments always.** Every migrated file carries a comment pointing to its source (or sources, for split content). This is non-negotiable.
- **Source files are never modified.** KEEP reads them; the user owns cleanup.
- **`/keep-observe` and `/keep-init` never write during ingestion classification.** They only propose the plan.
- **`/keep-compile` writes one file at a time with explicit approval.** No batch migration.
- **Heading signal beats keyword signal.** When the two disagree, prefer structure. When neither is conclusive, ask.
