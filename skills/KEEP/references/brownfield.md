# Brownfield adoption — cordon, don't ingest

Read this when `/keep-compile` is given a folder source (e.g. `./docs/`, `./old-docs/`), or when `/keep-init` ran and pre-existing docs were detected.

## The default behavior: cordon-off

KEEP v4 reversed an earlier policy. **The default treatment of pre-existing documentation is to cordon it off, not to ingest it.** Concretely: instead of converting old docs into specs/ADRs, `/keep-compile <folder>` writes a single ADR that:

- Declares the folder as **legacy / unverified** — out of scope for drift detection and `/keep-ask` synthesis.
- Lists the files for traceability.
- Sets the boundary: "from this point forward, `/knowledge/` is authoritative for new behavior; the legacy folder remains for historical context only."

This is the [Anchored Development](https://anchored-dev.org/getting-started/) recommendation: *"the documentation you already have… the README.md that describes last year's architecture, the design docs nobody updated, the Markdown monsters. These are unverified artifacts. They will lie to you, and they will lie to every AI agent that reads them. Consider cordoning off existing documentation with an ADR that acknowledges the migration, then move forward as if it were a new project."*

The reasoning is asymmetric: a wrong spec is worse than a missing spec. A spec that confidently describes last year's architecture poisons `/keep-ask` and silently misleads new contributors. Better to admit ignorance and grow the knowledge layer from real change going forward.

## What `/keep-compile <folder>` produces by default

```
your-repo/
├── docs/                    ← legacy, untouched, NOT migrated
│   ├── ARCHITECTURE.md
│   ├── runbooks/*.md
│   └── ...
└── knowledge/
    └── docs/decisions/
        └── ADR-NNNN-legacy-docs-cordoned.md  ← the cordon ADR
```

The cordon ADR's body, written verbatim:

```md
---
id: ADR-NNNN
title: "Legacy documentation cordoned off"
description: "Pre-existing documentation in <path> is declared legacy and out of scope for /keep-ask, /keep-check-drift, and INDEX. Retained for historical reference only."
status: accepted
type: adr
domain: keep
tags: [keep, brownfield, migration]
related:
  - code:<path>
---

# ADR-NNNN: Legacy documentation cordoned off

## Status
Accepted

## Context

This repository adopted KEEP on <YYYY-MM-DD>. Pre-existing documentation lived in `<path>` and may describe how the system worked at some earlier point in time. We do not have the bandwidth to validate each file against current code, and migrating unverified prose into the knowledge layer would poison `/keep-ask` (every answer would carry the implicit claim "this is current truth", which is unsupported).

## Decision

`<path>` is declared **legacy / out of scope** for KEEP. Concretely:

- `/keep-ask` does NOT load files from `<path>`.
- `/keep-check-drift` does NOT verify anchors against `<path>`.
- `INDEX.md` does NOT list `<path>` files.
- The folder is retained in the repo unchanged. Contributors may still read it for context.

## Alternatives considered

### Migrate all docs into `/knowledge/` with per-file approval
- Rejected: high effort, and each ingested file carries an implicit currency claim we cannot back. The "Markdown monster" risk dominates the labor cost.

### Delete the legacy folder
- Rejected: historical context has non-zero value; deleting is a one-way operation; the folder can be migrated piecemeal later if a specific file proves valuable.

## Consequences

- New behavior is documented in `/knowledge/` going forward, growing organically from `/keep-compile` on real diffs.
- A specific legacy doc can be migrated later by explicit user request (see "Opt-in selective migration" below). The default is *not migrated*.
- `/keep-ask` may return *"no indexed knowledge"* on topics covered only in the legacy folder. The agent must say so honestly rather than load and quote untrusted prose.

## Drivers

- Asymmetric risk: a wrong spec poisons every downstream query; a missing spec is just a gap.
- KEEP's value compounds when the knowledge layer is trustworthy. Cordoning protects trust from day one.
- Anchored Development's published guidance recommends this approach.

## Files cordoned (snapshot at adoption time)

- `<path>/ARCHITECTURE.md`
- `<path>/runbooks/jwt-rotation.md`
- ... (one line per file)
```

The file list is captured at adoption time as a historical record. Future files added to `<path>` are also legacy by inheritance unless explicitly migrated.

## Opt-in selective migration — read, verify, curate, write

A specific legacy file can be promoted into the knowledge layer with `--migrate`:

```
/keep-compile <path/to/specific-file.md> --migrate
```

`--migrate` is **not** a verbatim copy. It triggers a structured verification pass — the agent reads the file, extracts the durable claims, checks them against current code, and asks the user about anything that disagrees or has no binding. The resulting spec lands in `/knowledge/` carrying only facts that are still true today.

### The five steps of `--migrate`

1. **Classify** — propose target type (spec / spec+runbook / spec+architecture / ADR / idea) from the file's headings and body. Confirm with the user.

2. **Extract claims** — split the file's content into verifiable claims (literal values, function signatures, endpoints, test names, file references) and non-verifiable prose (rationale, history, alternatives considered).

3. **Verify** — for each verifiable claim, run a deterministic check against current code (`grep` for the symbol, compare value/signature). Mark each claim ✓ matches / ✗ contradicts / ? unverifiable.

4. **Ask** — batch the ?-rows into one turn of questions (max 3-4 at a time). For ✗-rows, propose three actions: drop the claim, replace with current truth, or keep with `<!-- TODO(KEEP): outdated -->` marker. Don't ask about ✓-rows — they're already verified.

5. **Write** — produce the spec with: verified claims as text *and* as anchor entries in frontmatter (the verification gave us bindings for free), corrected/dropped contradictions, user-decided handling of unverifiables. Add provenance comment near the top: `<!-- Migrated from <path> on <date>; verified against commit <sha>; claims confirmed/dropped per session with @user -->`.

### Why not verbatim?

A verbatim copy of an old doc carries the implicit claim *"this is the current truth"*. `/keep-ask` treats every file in `/knowledge/` as authoritative; a doc you migrated yesterday and a doc you wrote yesterday have the same authority unless you've done something to distinguish them. The verification pass forces that distinction *now*, instead of letting unverified prose silently masquerade as truth.

### Why single-file only?

A folder of 10+ unverified docs cannot be verified by hand in one session — by the time the operator finishes the verification interview on file 10, they've forgotten what file 1 said. Cordon-off the folder, then promote the 1-3 docs that are genuinely worth migrating. The friction is the feature; the resulting `/knowledge/` is small, intentional, and trusted.

### Classification heuristics (for `--migrate`)

These are the same heuristics earlier KEEP versions applied to every file by default. They now apply only when the user opts in per file.

Read the file's top-level + second-level headings (structural signal) and body keywords (semantic signal). When they disagree, **prefer headings**.

| Target type | Heading patterns | Keyword signals | Output path |
|---|---|---|---|
| **spec** (behavior) | `## Endpoint`, `## API`, `## Behavior`, `## Requirements`, `## Acceptance criteria`, `## Inputs/Outputs`, `## Validation`, `## Edge cases`, `## Errors`, `## Responses` | "shall", "must return", "given … when … then", "rejects", "validates", "returns 4xx/5xx", request/response examples | `specs/<package>/<slug>.md` |
| **spec + `runbook` tag** | `## Symptoms`, `## Detection`, `## Cause(s)`, `## Mitigation`, `## Rollback`, `## Postmortem`, `## Recovery`, `## Alerts` | "alert", "paged", "on-call", "incident", "post-mortem", "RTO/RPO", "SLO breach", "5xx spike", "OOMKilled", "circuit breaker" | `specs/<package>/<slug>.md` with `tags: [<domain>, runbook]` |
| **spec + `architecture` tag** | `## Topology`, `## Components`, `## Boundaries`, `## Data flow`, `## Sequence`, `## Diagram`, `## Overview` | "service", "boundary", "talks to", "depends on", "data flow", ASCII/mermaid diagrams | `specs/<package>/topology.md` with `tags: [<domain>, architecture]` |
| **ADR** | `## Status`, `## Context`, `## Decision`, `## Consequences`, `## Alternatives`, `## Considered options`, `## Drivers` | "we chose X because", "rejected Y", "instead of", "tradeoff", "we considered", explicit alternatives list | `decisions/ADR-NNNN-<slug>.md` |
| **idea** | Free-form, often labeled `## Proposal`, `## Idea`, `## Thinking`, `## RFC draft` | "we should", "what if", "thinking about", "not committed", "to investigate" | `ideas/<YYYY-MM-DD>-<slug>.md` with `status: draft` |
| **mixed / unclear** | Multiple structural patterns OR thin / boilerplate content | — | Ask the user to split or skip; never lump |

Infer `<package>` from the source path: `services/auth/docs/jwt.md` → `specs/auth/jwt.md`. In single-package repos use `<area>` derived from filename or top heading.

### Files never migrated, even with `--migrate`

- `CHANGELOG.md` — derived from git history
- `CONTRIBUTING.md`, `CODE_OF_CONDUCT.md` — process docs, not system knowledge
- Auto-generated API references — derived artifact; write a spec from scratch if behavior matters
- `LICENSE`, `NOTICE` — legal text, not knowledge

## Where to scan during adoption (for the cordon ADR's file list)

Walk these locations recursively, excluding `node_modules`, `.git`, `dist`, `build`, `target`, `vendor`, `.venv`, and anything under `/knowledge/`:

- `README.md` at repo root and inside each package
- `docs/`, `doc/`, `documentation/` (any depth)
- `ARCHITECTURE.md`, `ARCHITECTURE/`, `DESIGN.md`, `DESIGN/`
- `RUNBOOK.md`, `RUNBOOKS/`, `runbook/`, `runbooks/`
- `notes/`, `design-notes/`, `rfc/`, `rfcs/`, `adr/`, `decisions/`
- `wiki/`, `.wiki/`
- Top-level `*.md` excluding the boilerplate above

The scan produces the file list that goes into the cordon ADR's body. It does NOT propose migration of those files — the whole point of cordon-off is that the default is "don't touch".

## Hard rules

- The default for a folder source to `/keep-compile` is **write one cordon ADR, not ingest**.
- Migration of a specific file requires `--migrate` and per-file approval. No batch migration.
- Source files are read-only. KEEP never modifies, moves, or deletes them.
- `/keep-ask` and `/keep-check-drift` ignore everything under a cordoned path.
- Re-running `/keep-compile <same-folder>` is idempotent: if a cordon ADR already exists for that path, it updates the file list and timestamp instead of writing a duplicate.
