# File formats and templates

Templates for every file type in `/knowledge`, plus three cross-cutting conventions: **mandatory YAML frontmatter**, **linking between files**, and **ADR supersession**.

Why the frontmatter is mandatory: it powers everything else. `/keep-ask` filters by `description`, `tags`, and `domain` before opening files. `/keep-check-drift` matches changed code to affected knowledge via `related` patterns. `INDEX.md` is regenerated from frontmatter by `scripts/build_index.py` — files without frontmatter end up in a "needs fixing" section and are invisible to retrieval. If you find yourself writing a knowledge file without frontmatter, you are creating an unverified artifact that will drift and lie. Don't.

## Cross-cutting convention 1 — Mandatory YAML frontmatter

Every file under `/knowledge/docs/` and `/knowledge/ideas/` must start with this block:

```yaml
---
id: SPEC-auth-jwt              # stable identifier — type-domain-slug, immutable once published
title: "JWT validation"        # short human title
description: "Behavioral spec for JWT issuer/audience validation, expiry handling, and the refresh flow on /auth/* endpoints. Covers TTL, signature rotation, error cases."
status: accepted               # draft | accepted | deprecated | superseded
type: spec                     # spec | adr | idea  (runbook/architecture are tags on specs)
domain: auth                   # cohesive area; matches a top-level subdirectory under specs/
tags: [auth, jwt, security]    # free-form, lowercase, hyphens. Use 'runbook' or 'architecture' as tag when relevant.
related:
  - adr:ADR-0014
  - test:internal/auth/*_test.go matching TestJWT*
  - code:internal/auth/jwt.go
---
```

### Field semantics

- **`id`** — stable, immutable once published. ADRs use `ADR-NNNN` (zero-padded). Specs use `SPEC-<domain>-<slug>`. Ideas use `IDEA-<YYYY-MM-DD>-<slug>`. The id is the canonical handle for cross-references and citation in `/keep-ask` answers.
- **`title`** — short human-readable title. Used in tables and as the H1 of the body.
- **`description`** — **the most load-bearing field.** Written as a *search snippet*, not a chapter heading. `/keep-ask` decides whether to load a file based on this string matching the user's question. Bad: `"Worker stuff"`. Good: `"Behavioral expectations for worker registration, heartbeat, job assignment, shutdown, and disconnection handling"`. Include the keywords a future reader would type.
- **`status`** — `draft` (proposed, not validated), `accepted` (authoritative), `deprecated` (still in repo for context, but the team is moving away), `superseded` (replaced by a specific newer file; the supersession protocol below applies).
- **`type`** — strict vocabulary. `spec` covers what the system should do (behavioral). `adr` covers a decision with rejected alternatives. `idea` covers half-formed proposals in `/knowledge/ideas/`. Runbooks and architecture docs are *specs with a tag*, not separate types — this is intentional, see below.
- **`domain`** — must match a real area of the system. In a monorepo, this is the package name (`auth-service`, `inference-api`). In a single-package repo, a logical area (`auth`, `billing`). Used to scope `/keep-ask` and `/keep-check-drift`.
- **`tags`** — free-form keywords. Two tags are *semantically reserved*: `runbook` marks a spec that documents failure modes/operational behavior; `architecture` marks a spec that describes topology. Use them so `INDEX.md` can group correctly.
- **`related`** — convention-based cross-references. Prefixes are mandatory: `adr:`, `spec:`, `test:`, `code:`, `runbook:` (a tag-runbook spec), `idea:`. For code/test references, use *path patterns or naming patterns*, never hard-coded line numbers — those break the moment a file is renamed. See the example: `test:internal/auth/*_test.go matching TestJWT*` is a pattern, not a list.

### Why no separate `runbook` and `architecture` types

KEEP v1 had four types: spec, adr, runbook, architecture. The first three months of usage showed they were the same artifact wearing different hats — all behavioral descriptions, all consulted the same way, all under the same retrieval logic. Forcing four directories created a routing decision on every write ("is this a runbook or a spec?") with no real downstream benefit.

The v2 collapse: `spec` is the artifact type; `runbook` and `architecture` are tags that change which sections the template emphasizes. The retrieval pipeline doesn't care; the human reading the spec sees a Symptoms/Causes section because the spec carries the `runbook` tag. Less vocabulary, same expressiveness.

If you want a runbook-style file: `type: spec`, `tags: [runbook, ...]`, body follows the runbook template below. Same for architecture. `INDEX.md` auto-generates dedicated sections for both based on the tags.

## Cross-cutting convention 2 — Linking between files

The `related:` block in the frontmatter is the *machine-readable* cross-reference. For prose, you can also add a `## Related` section at the bottom of the body with relative markdown links:

```md
## Related

- Implements: [ADR-0014 — Ray Serve for inference](../decisions/ADR-0014-ray-serve.md)
- Operationalized by: [specs/auth/jwt-rotation.md](./jwt-rotation.md)
```

Use **relative paths** from the file's location. The relationship verb (`Implements`, `Supersedes`, `Operationalized by`, `Depends on`, `See also`, `Refines`) is free-form but should be informative.

When `/keep-compile` updates a file, it should update the `## Related` section and the *other side* of the link (if spec A now implements ADR B, both files should reference each other).

## Cross-cutting convention 3 — ADR supersession

Decisions get revisited. The supersession protocol below preserves history while making the current state of the world unambiguous.

### When ADR-MMMM supersedes ADR-NNNN

1. **Create the new ADR (MMMM) normally** with `status: accepted`. In `related:`, add `adr:ADR-NNNN` with a `supersedes` note. In the **Context** section, explain *what changed* that made the old decision obsolete.

2. **Update the old ADR (NNNN) in place.** Two changes only:
   - Set `status: superseded` in the frontmatter.
   - Append a `## Superseded by` section at the end with a one-line "what changed" summary and a link to ADR-MMMM.
   - **Never edit the body of a superseded ADR.** It is a historical record.

3. **INDEX.md regenerates automatically.** `scripts/build_index.py` moves superseded ADRs into a dedicated section based on their status. You do not edit the index manually.

### Partial supersession (refinement)

When a new ADR doesn't fully replace an old one — it modifies a specific aspect — use `Refines:` in the prose `## Related` section and keep the old ADR's status as `accepted`. The old ADR adds a `## Refined by` section pointing to the newer file. Confusing refinement with supersession loses valid context.

### Translating user statements about status

| User says... | Status |
|---|---|
| "still accepted" / "still current" | `accepted` |
| "this has been replaced" / "we don't do this anymore" | `superseded` (with full supersession protocol) |
| "partially superseded" / "we updated part of this" | `accepted` + `Refines:` link from the newer ADR |
| "deprecated" / "we're moving away from this" | `deprecated` |
| "we never really followed this" / "this was aspirational" | `deprecated` |

If the user names the successor decision, capture it. If they don't, do not fabricate — leave a `<!-- TODO(KEEP): successor decision not identified -->` marker and let `/keep-govern` surface it later.

---

## Specs — `/knowledge/docs/specs/<domain>/<feature>.md`

Specs describe *intended behavior*, not implementation. They evolve as the system evolves.

```md
---
id: SPEC-auth-jwt
title: "JWT validation"
description: "Behavioral expectations for JWT validation: issuer/audience checks, expiry, secret rotation."
status: accepted
type: spec
domain: auth
tags: [auth, jwt, security]
related:
  - adr:ADR-0014
  - test:internal/auth/*_test.go matching TestJWT*
  - code:internal/auth/jwt.go
---

# JWT validation

## Goal
<one or two sentences>

## Requirements
- <requirement>
- <requirement>

## Edge cases
- <case>
- <case>

## Acceptance criteria
- <verifiable criterion>
- <verifiable criterion>
```

Keep specs minimal. If a section is empty, omit it.

### Spec with `runbook` tag (operational knowledge)

A spec tagged `runbook` describes failure modes and operational response, not desired behavior. The body uses these sections instead:

```md
---
id: SPEC-auth-jwt-rotation
title: "JWT secret rotation"
description: "Failure mode and operational runbook for JWT secret rotation: 401 spike symptoms, dual-secret mitigation, prevention strategy."
status: accepted
type: spec
domain: auth
tags: [auth, runbook, jwt]
related:
  - spec:SPEC-auth-jwt
---

# JWT secret rotation

## Symptoms
- <observable signal>

## Causes
- <root cause>

## Mitigation
- <step>

## Prevention
- <change that would prevent recurrence>
```

Only write a runbook for a failure that has actually happened or is genuinely likely. Speculative runbooks become noise.

### Spec with `architecture` tag (topology / boundaries)

A spec tagged `architecture` describes topology, components, boundaries. The body uses these sections:

```md
---
id: SPEC-auth-architecture
title: "Auth service architecture"
description: "Topology and boundaries of the auth service: components, data flow, dependencies on Postgres, what is intentionally outside."
status: accepted
type: spec
domain: auth
tags: [auth, architecture]
related:
  - adr:ADR-0001
---

# Auth service architecture

## Components
- <component>: <one-line role>

## Flow
<text or ASCII diagram>

## Boundaries
- <what is inside>
- <what is intentionally outside>

## Constraints
- <constraint>
```

ASCII diagrams are fine and often more durable than image links:

```
Frontend → API Gateway → Auth Service → Postgres
                              ↓
                         JWT (HS256)
```

## ADRs — `/knowledge/docs/decisions/ADR-NNNN-<slug>.md`

Architecture Decision Records capture *why* a non-trivial choice was made and what alternatives were rejected.

```md
---
id: ADR-0014
title: "Ray Serve for ML inference"
description: "Adopted Ray Serve for the inference layer. KServe was rejected due to CRD complexity. Custom FastAPI was rejected as undifferentiated reinvention of autoscaling and batching."
status: accepted
type: adr
domain: inference
tags: [inference, infrastructure, ml]
related:
  - adr:ADR-0007  # supersedes
  - spec:SPEC-inference-pipeline
  - code:services/inference/server.go
---

# ADR-0014: Ray Serve for ML inference

## Status
Accepted

## Context
<the problem or constraint that forced a choice>

## Decision
<what was chosen, in one or two sentences>

## Alternatives considered
### <alternative 1>
- <why rejected>

### <alternative 2>
- <why rejected>

## Consequences
- <tradeoff>
- <tradeoff>

## Drivers
<the positive factors that made this option specifically right, distinct from rejected-alternatives reasoning>
- <factor>
- <factor>
```

### ADR numbering

Sequential and never reused. **Before assigning a number, list the existing `decisions/` directory** and use the next available integer — never pre-number from memory or from the conversation flow.

### The three rationale sections

- **Alternatives considered** answers *"why didn't we pick X?"* — one entry per rejected option with its specific rejection reason.
- **Consequences** answers *"what cost are we accepting by picking this?"* — tradeoffs, downsides, future obligations.
- **Drivers** answers *"why this option specifically?"* — the positive factors that made it win on its own merits.

Don't merge these into one section. The separation is what makes ADRs useful in a year when someone asks "what were we thinking?".

## Ideas — `/knowledge/ideas/<slug>.md`

Half-formed proposals not yet ready for ADR or spec. Captured durably so they don't disappear in chat history.

```md
---
id: IDEA-2026-05-13-jwt-grace-window
title: "Grace window for JWT secret rotation"
description: "Idea: introduce a grace window where both current and previous JWT secrets validate, to avoid the 401 spike at deploy time."
status: draft
type: idea
domain: auth
tags: [auth, jwt, rotation, brainstorm]
related:
  - runbook:SPEC-auth-jwt-rotation
created: 2026-05-13
---

# Grace window for JWT secret rotation

## Context
<why this came up, what triggered the idea>

## Sketch
<the rough proposal — not a final design>

## Open questions
- <question to resolve before promoting>
- <question to resolve before promoting>
```

Status flow: `draft` → either promoted into a spec/ADR (via `/keep-compile`) or marked `deprecated` if dropped. `/keep-govern` flags ideas older than 30 days that haven't moved.

## INDEX.md — auto-generated, not hand-maintained

`INDEX.md` is regenerated from frontmatter by `scripts/build_index.py`. It is the entry point for `/keep-ask` — the command reads it first to decide which files to open.

**Do not edit `INDEX.md` by hand.** The script:

1. Walks `/knowledge/docs/` and `/knowledge/ideas/`.
2. Parses YAML frontmatter from each file.
3. Validates required fields (`id`, `title`, `description`, `status`, `type`, `domain`, `tags`).
4. Renders one table per `type`, sorted by domain then id.
5. Aggregates `runbook` and `architecture` tags into their own sections.
6. Moves `superseded` entries into a dedicated section at the bottom.
7. Flags files with invalid/missing frontmatter into an "issues" section.

`/keep-compile` calls the script as its last step. You should never write to `INDEX.md` directly — if the schema produces a layout you don't like, fix the script, not the output.

### What you get out

A single, deterministic, table-based index that:

- Lets the agent decide which files to load by reading 200 tokens (the table) instead of every spec body.
- Makes "all files tagged X" or "all files in domain Y" trivial filters.
- Surfaces frontmatter issues as a visible warning instead of silent index corruption.
- Eliminates the v1 "Entities" / "Flows" sections that conflated semantically distinct things.

