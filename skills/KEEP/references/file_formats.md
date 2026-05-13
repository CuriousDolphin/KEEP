# File formats and templates

Templates for every file type in `/knowledge`, plus two cross-cutting conventions: **linking between files** and **ADR supersession**. Use the templates as starting points, not rigid molds — adapt to the content but stay close to the structure so files are easy to scan.

## Cross-cutting convention 1 — Linking between files

Knowledge files reference each other constantly. A spec implements a decision. An ADR affects an architecture component. A runbook documents a failure mode of a particular spec. Without explicit links, these relationships exist only in the reader's head, and `/keep-govern` cannot detect contradictions.

### The convention

Each file type has an optional `## Related` section at the bottom listing cross-references. Inline links inside the body are also fine when the connection is natural in context.

```md
## Related
- Implements: [ADR-0014 — Ray Serve for inference](../decisions/ADR-0014-ray-serve.md)
- Operationalized by: [runbooks/ray-serve-restart.md](../runbooks/ray-serve-restart.md)
- See also: [architecture/inference.md](../architecture/inference.md)
```

Use **relative paths** from the file's location. The relationship verb (`Implements`, `Supersedes`, `Operationalized by`, `Depends on`, `See also`) is free-form but should be informative.

### Relationship vocabulary

Reusable labels that work well:

| From | To | Label |
|---|---|---|
| Spec | ADR | `Implements:` |
| Spec | Architecture | `Component of:` |
| ADR | ADR | `Supersedes:` / `Superseded by:` / `Refines:` |
| ADR | Architecture | `Shapes:` |
| Runbook | Spec | `Operationalizes:` |
| Runbook | ADR | `Consequence of:` |
| Architecture | ADR | `Documented by:` |

When `/keep-compile` updates a file, it should add or update the `## Related` section based on the changes — and update the *other side* of the link too. If spec A now implements ADR B, both files should reference each other.

---

## Cross-cutting convention 2 — ADR supersession

Decisions get revisited. The supersession protocol below preserves history while making the current state of the world unambiguous.

### When ADR-MMMM supersedes ADR-NNNN

1. **Create the new ADR (MMMM) normally.** Its first relationship line is:
   ```md
   ## Related
   - Supersedes: [ADR-NNNN — <old title>](./ADR-NNNN-<old-slug>.md)
   ```
   In the **Context** section, explain *what changed* that made the old decision obsolete. This is the load-bearing part — without it the supersession looks arbitrary.

2. **Update the old ADR (NNNN) in place.** Do not delete it. Change two things:
   - Status line at the top:
     ```md
     ## Status
     Superseded by [ADR-MMMM — <new title>](./ADR-MMMM-<new-slug>.md) on YYYY-MM-DD
     ```
   - Add at the end of the file, preserving everything above:
     ```md
     ## Superseded by
     [ADR-MMMM — <new title>](./ADR-MMMM-<new-slug>.md)

     ### What changed
     <one or two sentences from the new ADR's context, summarizing why this decision no longer holds>
     ```

3. **Never edit the body of a superseded ADR** — it is a historical record. The only edits are the status line and the appended "Superseded by" section.

4. **Update `INDEX.md`** to note the superseded ADR under a `## Superseded decisions` heading (or equivalent) so it doesn't clutter the live decision list but remains discoverable.

### Partial supersession (refinement)

When a new ADR doesn't fully replace an old one — it modifies a specific aspect — use `Refines:` instead of `Supersedes:`. The old ADR keeps `Status: Accepted` but adds a `## Refined by` section pointing to the newer file. This pattern matters: confusing refinement with supersession leads to losing valid context.

### Translating user statements about status

When the user describes the status of an ADR in natural language, map their words to the KEEP patterns rather than inventing ad-hoc Status values. This translation table is canonical:

| User says... | KEEP pattern |
|---|---|
| "still accepted" / "still current" / "this is how we do it" | `Status: Accepted` |
| "this has been replaced" / "we don't do this anymore" | `Status: Superseded by ADR-MMMM` + full supersession protocol |
| "partially superseded" / "mostly current but X changed" / "we updated part of this" | `Status: Accepted` + a `## Refined by` section pointing to the newer ADR (use the `Refines:` pattern above) |
| "deprecated" / "we're moving away from this" | `Status: Deprecated` (with a sentence explaining the migration direction) |
| "we never really followed this" / "this was aspirational" | `Status: Deprecated` (and note this honestly in the body) |

If the user names the successor decision, capture it. If they don't, do not fabricate one — write the appropriate Status, add a `<!-- TODO(KEEP): successor decision not identified at compile time -->` marker, and let `/keep-govern` surface it later. **Never** invent a custom Status value like *"partially superseded but mostly current"* — those are signals that the standard patterns apply, you just need to apply them.

---

## Specs — `/knowledge/docs/specs/<area>/<feature>.md`

Specs describe *intended behavior*, not implementation. They evolve as the system evolves.

```md
# <Feature name>

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

## Related
- Implements: [ADR-NNNN — <title>](../../decisions/ADR-NNNN-<slug>.md)
- Operationalized by: [runbooks/<failure-mode>.md](../../runbooks/<failure-mode>.md)
```

Keep specs minimal. If a section is empty, omit it — do not pad. `## Related` is optional but encouraged whenever a spec is the operational arm of an ADR.

## ADRs — `/knowledge/docs/decisions/ADR-NNNN-<slug>.md`

Architecture Decision Records capture *why* a non-trivial choice was made and what alternatives were rejected.

```md
# ADR-NNNN: <decision title>

## Status
Accepted | Superseded by [ADR-MMMM — <title>](./ADR-MMMM-<slug>.md) on YYYY-MM-DD | Deprecated

## Scope
<optional — when the ADR's scope isn't obvious. Especially useful in a monorepo
to state which packages the decision applies to. Omit if scope is clear from the title.>

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
<the positive factors that made the chosen option specifically the right one.
This is distinct from Consequences (which are tradeoffs accepted as a cost) and
distinct from Alternatives considered (which are why others lost). Drivers are
why this option won on its own merits.>

- <factor>
- <factor>

## Related
- Supersedes: [ADR-NNNN — <title>](./ADR-NNNN-<slug>.md)        # if applicable
- Refines: [ADR-NNNN — <title>](./ADR-NNNN-<slug>.md)           # if partially supersedes
- Shapes: [architecture/<area>.md](../architecture/<area>.md)
- Implemented by: [specs/<area>/<feature>.md](../specs/<area>/<feature>.md)
```

Only create an ADR when there are real rejected alternatives. A choice with no alternatives is not a decision; it is just a fact, and belongs in a spec or architecture doc.

Numbering is sequential and never reused. Before assigning a number, list the existing `decisions/` directory and use the next available integer — never pre-number from memory. The supersession protocol above governs status transitions.

**On the three rationale sections.** ADRs have three sections that each carry a different kind of rationale, and it's worth being clear on what goes where:

- **Alternatives considered** answers *"why didn't we pick X?"* — one entry per rejected option with its specific rejection reason.
- **Consequences** answers *"what cost are we accepting by picking this?"* — tradeoffs, downsides, future obligations.
- **Drivers** answers *"why this option specifically?"* — the positive factors that made it win on its own merits, independent of what was rejected.

When the user describes their decision, you often hear answers that fit naturally into one of these three. Don't merge them into a single section; the separation is what makes ADRs useful in a year when someone asks "what were we thinking?".

## Architecture — `/knowledge/docs/architecture/<area>.md`

Describe topology, components, boundaries, and data flow. Avoid implementation details.

```md
# <System area>

## Components
- <component>: <one-line role>
- <component>: <one-line role>

## Flow
<text or ASCII diagram showing the path of data/control>

## Boundaries
- <what is inside this area>
- <what is outside, and which area owns it>

## Constraints
- <constraint that shapes the architecture>

## Related
- Documented by: [ADR-NNNN — <title>](../decisions/ADR-NNNN-<slug>.md)
```

ASCII diagrams are fine and often more durable than image links:

```
Frontend → API Gateway → Ray Serve → Triton
                              ↓
                          Postgres
```

## Runbooks — `/knowledge/docs/runbooks/<failure-mode>.md`

Operational knowledge derived from real production experience. Not theoretical.

```md
# <Failure mode>

## Symptoms
- <observable signal>
- <observable signal>

## Causes
- <root cause>
- <root cause>

## Mitigation
- <step>
- <step>

## Prevention
- <change that would prevent recurrence, if applicable>

## Related
- Operationalizes: [specs/<area>/<feature>.md](../specs/<area>/<feature>.md)
- Consequence of: [ADR-NNNN — <title>](../decisions/ADR-NNNN-<slug>.md)
```

Only create a runbook for a failure that has actually happened or is genuinely likely. Speculative runbooks become noise.

## INDEX.md — incrementally maintained by `/keep-compile`

`INDEX.md` is a flat semantic map of the knowledge base. It is the entry point for `/keep-retrieve`.

```md
# INDEX

## Domains

### <Domain name, e.g. Auth>
- knowledge/docs/specs/auth/jwt.md
- knowledge/docs/decisions/ADR-0001-auth0.md
- knowledge/docs/runbooks/jwt-rotation.md

### <Domain name, e.g. Inference>
- knowledge/docs/architecture/inference.md
- knowledge/docs/specs/inference/pipeline.md

## Entities
- JWT
- Auth0
- Ray Serve
- Triton

## Flows
- authentication flow
- inference pipeline

## Superseded decisions
- ADR-0007-kserve.md → superseded by ADR-0014-ray-serve.md
```

### How to update `INDEX.md` incrementally

`/keep-compile` does **not** regenerate the entire `INDEX.md` on every run. It does the following:

1. **Identify affected sections.** Which domain(s) does this `/keep-compile` touch? Which new entities, if any, are introduced? Did any flow change?
2. **Modify only those sections.** Insert new file paths in the right domain. Add new entities to the entity list, deduplicating against existing ones. Add a flow only if it crosses component boundaries described in architecture files.
3. **Handle supersessions.** When an ADR is superseded, move its entry from the active domain list into the `## Superseded decisions` section at the bottom.
4. **Preserve everything else byte-for-byte.** Sections not affected by this `/keep-compile` should not appear in the diff.

### When to fully (re)generate `INDEX.md` instead

Full regeneration is the right move only in these specific situations:

- **First populate.** `INDEX.md` is missing entirely, is an empty file, or contains only a stub (less than ~5 non-empty lines — basically just a `# INDEX` heading and nothing else). In this case there is nothing meaningful to preserve, and regeneration produces a clean baseline.
- **Explicit user request.** The user invokes `/keep-compile --rebuild-index` or asks in plain language to rebuild the index from scratch.
- **Detected corruption.** The existing `INDEX.md` references files that no longer exist, or its structure has drifted so far from the current `/knowledge/docs` tree that incremental updates would patch over broken state. In this case, surface the problem to the user and ask before regenerating — do not silently rebuild.

For every other run, incremental is the default. A full regeneration on a healthy index is destructive: it loses any manual edits the user made, reshuffles the order of entries (creating noise in git diffs), and may drop entities or flows whose source is not currently being touched.

### How to extract entities and domains

When updating `INDEX.md`:

- **Domains** come from the top-level subdirectories under `specs/`, `decisions/`, `runbooks/`, and `architecture/`. Group files by domain.
- **Entities** are proper nouns repeated across multiple files: service names, technologies, key concepts. Deduplicate aggressively.
- **Flows** are named processes that span multiple components.

Do not invent entities or flows that are not actually mentioned in the knowledge files. The index reflects what is documented, not what could be.

### Single-entry domains

A domain with only one file is fine — it just means knowledge in that area is still nascent. List it normally; do not invent siblings to make it look fuller, and do not hide it. As more files accumulate the domain section will fill in naturally.

The one case worth flagging is when many domains end up with only one entry each. That suggests the domains are too fine-grained; consider collapsing related single-entry domains into one broader heading on the next `/keep-compile` (with user approval). `/keep-govern` should surface this pattern when it detects more than ~5 single-entry domains in the same INDEX.

## Tasks — `/knowledge/tasks/<task-name>.md`

Ephemeral execution state. Free-form body with a minimal YAML frontmatter that `/keep-compile` manages automatically.

```md
---
status: active        # active | done | abandoned
created: 2026-05-12
topic: auth-refresh   # short slug matching a domain or feature
---

# Auth refresh — implementation notes

<free-form notes — plans, scratch work, debug traces, whatever helps>
```

Field semantics:

- **`status`** — `active` (work in progress), `done` (work concluded, durable knowledge updated), `abandoned` (work dropped). Maintained by `/keep-compile`.
- **`created`** — ISO date the task was created. Set once, never modified.
- **`topic`** — short slug, ideally matching a domain in `INDEX.md` or a feature name. Used by `/keep-compile` to correlate tasks with knowledge files.

The user does not maintain this frontmatter by hand — `/keep-compile` and `/keep-govern` are the canonical writers. If a task file is created without frontmatter, `/keep-compile` inserts a default block on first touch.

**Archival convention:** when a task transitions to `done` or `abandoned`, it is moved to `/knowledge/tasks/_archive/<YYYY-MM-DD>-<original-name>.md` where the date comes from the `created` field. `active` tasks are never archived. See the task lifecycle section in `SKILL.md`.

Tasks are explicitly *not* durable knowledge and are never indexed in `INDEX.md`.
