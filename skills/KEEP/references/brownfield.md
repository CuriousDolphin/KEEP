# Brownfield ingestion — heuristic catalog

Read this when `/keep-compile` is invoked with a folder source (e.g. `./old-docs/`) or when `init.sh` surfaced pre-existing docs at adoption time.

## Where to scan

Walk these locations recursively, excluding `node_modules`, `.git`, `dist`, `build`, `target`, `vendor`, `.venv`, and any path already under `/knowledge/`:

- `README.md` at repo root and inside each package
- `docs/`, `doc/`, `documentation/` (any depth)
- `ARCHITECTURE.md`, `ARCHITECTURE/`, `DESIGN.md`, `DESIGN/`
- `RUNBOOK.md`, `RUNBOOKS/`, `runbook/`, `runbooks/`
- `notes/`, `design-notes/`, `rfc/`, `rfcs/`, `adr/`, `decisions/`
- `wiki/`, `.wiki/`
- Top-level `*.md` excluding boilerplate (`LICENSE`, `CONTRIBUTING`, `CODE_OF_CONDUCT`, `CHANGELOG`)

When `/keep-compile` runs against a git diff (not a folder), also peek at paths adjacent to the touched files — the same directory and one level up. Latent docs near the diff often carry rationale that the commits don't.

## Classification — two signals applied jointly

Read each candidate's top-level + second-level headings (structural signal) and body keywords (semantic signal). When they disagree, **prefer headings** — structure is more reliable than wording.

| Target type | Heading patterns | Keyword signals | Path |
|---|---|---|---|
| **spec** (behavior) | `## Endpoint`, `## API`, `## Behavior`, `## Requirements`, `## Acceptance criteria`, `## Inputs/Outputs`, `## Validation`, `## Edge cases`, `## Errors`, `## Responses` | "shall", "must return", "given … when … then", "rejects", "validates", "returns 4xx/5xx", request/response examples | `specs/<package>/<slug>.md` |
| **spec + `runbook` tag** | `## Symptoms`, `## Detection`, `## Cause(s)`, `## Mitigation`, `## Rollback`, `## Postmortem`, `## Recovery`, `## Alerts` | "alert", "paged", "on-call", "incident", "post-mortem", "RTO/RPO", "SLO breach", "5xx spike", "OOMKilled", "circuit breaker" | `specs/<package>/<slug>.md` with `tags: [<domain>, runbook]` |
| **spec + `architecture` tag** | `## Topology`, `## Components`, `## Boundaries`, `## Data flow`, `## Sequence`, `## Diagram`, `## Overview` | "service", "boundary", "talks to", "depends on", "data flow", ASCII/mermaid diagrams, component lists | `specs/<package>/topology.md` (or `specs/shared/overview.md` for cross-cutting) with `tags: [<domain>, architecture]` |
| **ADR** | `## Status`, `## Context`, `## Decision`, `## Consequences`, `## Alternatives`, `## Considered options`, `## Drivers` | "we chose X because", "rejected Y", "instead of", "tradeoff", "we considered", explicit alternatives list | `decisions/ADR-NNNN-<slug>.md` (flat, sequential) |
| **idea** | Free-form, often labeled `## Proposal`, `## Idea`, `## Thinking`, `## RFC draft` | "we should", "what if", "thinking about", "not committed", "to investigate" | `ideas/<YYYY-MM-DD>-<slug>.md` with `status: draft` |
| **unclear / mixed** | Multiple structural patterns OR thin / boilerplate content | — | flag for user-driven split or rejection |

Infer `<package>` from the source path: `services/auth/docs/jwt.md` → `specs/auth/jwt.md`. In single-package repos use `<area>` derived from filename or top heading.

## Mixed-content sources

Common pattern: a single `ARCHITECTURE.md` containing high-level topology + per-component specs + a runbook for the deployment pipeline. Do NOT migrate it whole. Output a **split proposal** that the user approves chunk by chunk:

```
ARCHITECTURE.md  → split proposal:
  ├─ specs/shared/topology.md          (lines 1–48, tag: architecture)
  ├─ specs/auth/overview.md            (lines 50–120, tag: architecture)
  └─ specs/shared/deploy-pipeline.md   (lines 122–200, tag: runbook)
```

Quote line ranges from the source verbatim — KEEP does NOT rewrite human prose during ingestion. Restructuring is limited to:

- Adding YAML frontmatter at the top
- Inserting a `<!-- Migrated from <source> on YYYY-MM-DD, lines N–M -->` comment
- Stripping or fixing obvious typos that the author would also fix
- Reordering sections to match the relevant template (only when the original is clearly out of order)

## Per-file approval

Never migrate a folder in one shot. For each candidate, `/keep-compile` proposes the target type + path and asks for approval. The user can:

- **Accept** → file is migrated as proposed
- **Reject** → source is left in place, marked as "intentionally not in /knowledge"
- **Reclassify** → user picks a different target type
- **Split** → user marks line ranges and target types

The source file is never deleted. Migration is additive — the original lives where it is, and `/knowledge/` carries the canonical version with the provenance comment pointing back.

## When NOT to ingest

- **CHANGELOG.md** — derived from git history, not durable knowledge
- **CONTRIBUTING.md** / **CODE_OF_CONDUCT.md** — process docs, not system knowledge
- **API reference auto-generated from code** — derived artifact; if the user wants behavior captured, write a spec from scratch
- **Outdated docs the user explicitly marks as deprecated** — leave them in place with a note, don't migrate noise

## Hard rules

- Never migrate without per-file approval. Batch migration always produces noise.
- Quote source content verbatim during migration. KEEP is not a doc rewriter.
- Source files are read-only. KEEP never modifies, moves, or deletes them.
- Mixed-content sources require a split proposal, not a lump migration.
- After migration, run `scripts/build_index.py` so the new files appear in `INDEX.md`.
