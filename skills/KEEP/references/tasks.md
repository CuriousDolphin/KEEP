# Task lifecycle — details and operational rules

The summary is in `SKILL.md`. This file covers the operational specifics: how `/keep-compile` decides what to promote, how to handle tasks that span multiple `/keep-compile` runs, the exact frontmatter rules, and the edge cases.

## Frontmatter — full specification

Every task file in `/knowledge/tasks/` carries a minimal YAML frontmatter at the top:

```md
---
status: active        # active | done | abandoned
created: 2026-05-12
topic: auth-refresh
---

# Auth refresh — implementation notes

<free-form body — plans, scratch, debug traces, whatever helps execution>
```

### Field semantics

| Field | Values | Set by | Mutable |
|---|---|---|---|
| `status` | `active`, `done`, `abandoned` | `/keep-compile` | yes — transitions only |
| `created` | ISO date (YYYY-MM-DD) | task creation | never |
| `topic` | short slug (lowercase, hyphens) | task creation or `/keep-compile` | rarely, only on user request |

### Status transitions

The only allowed transitions, set by `/keep-compile`:

```
active → done           (work concluded successfully, durable knowledge updated)
active → abandoned      (work dropped without producing durable knowledge)
done → active           (work was prematurely closed and needs to continue)
abandoned → active      (work resumed after being dropped)
```

Direct `done ↔ abandoned` transitions are not allowed — they would lose information. Re-open via `active` first, then re-close to the correct status.

### Default frontmatter for files missing it

If a task file is found in `/knowledge/tasks/` without frontmatter, `/keep-compile` inserts a default block on first touch:

```yaml
---
status: active
created: <today>         # best guess, since the real creation date is unknown
topic: <inferred>        # from filename or content; ask if unclear
---
```

The `created` date in this case is the date `/keep-compile` ran, not the actual creation date — there's no reliable way to recover the latter. Document this in a comment if helpful: `# created: 2026-05-12  # filled in by /keep-compile on first touch`.

## Task creation

Tasks are created freely during work. No template is enforced for the body — whatever helps the current execution.

**Filename convention:** `<topic>-<verb>.md`. Examples:

- `auth-refresh-implementation.md`
- `inference-pipeline-debugging.md`
- `db-migration-checklist.md`

The convention is encouraged but not enforced. The `topic` field in frontmatter is the canonical source of truth for task-knowledge correlation.

## Promotion — how `/keep-compile` decides what to promote

When `/keep-compile` runs, it identifies task files whose `topic` matches the current change (or that the user explicitly references). For each matched task, scan the body for content that looks like durable knowledge.

### Signals that indicate promotable content

- **Decision rationale.** Phrases like "we chose X because", "tried Y, didn't work", "decided against Z", "alternatives considered" → ADR material.
- **Edge cases discovered during work.** "Turns out the API returns null when…", "Have to handle the case where the user…", "If the cache misses we have to…" → spec material.
- **Observed failure modes.** "Got a 502 because", "ran out of memory at batch size", "kept getting rate-limited" → runbook material.
- **Architectural insights.** "We need a new layer between X and Y", "this should be moved out of the API into a separate service" → architecture material.

### Signals that indicate non-promotable content (leave in tasks)

- Step-by-step checklists for one-off migrations.
- Debug logs and traces that don't generalize.
- Personal scratch ("try this", "ask Bob about that", "look up Z").
- TODOs and reminders that are specific to the current work and not durable behavior.

### Surfacing candidates to the user

For each promotable chunk, surface it explicitly with a target file:

> *"`auth-refresh-implementation.md` contains a note about why we chose sliding-window expiration over fixed-window. Quote:*
> *> 'Fixed-window felt simpler but it would have caused every user to be logged out at the same minute every day. Sliding-window distributes the load.'*
> *This looks like rationale for ADR-0014. Promote it into the ADR's Context or Alternatives section?"*

The user accepts, rejects, or asks to edit the wording. Migration happens only with explicit approval.

### Migration mechanics

When the user approves promotion:

1. **Copy, don't move.** The content goes into the durable file. The original task body is preserved as-is (it might contain additional context that didn't quite belong in the durable doc).
2. **Apply elicitation.** If the promoted content is incomplete (e.g. one alternative discussed, but the ADR template wants three), batch-ask for the gaps.
3. **Quote verbatim where possible.** Especially for rationale — the user's exact wording carries more weight than a paraphrase.
4. **Add a source comment** in the durable file pointing back to the task:
   ```md
   <!-- Promoted from /knowledge/tasks/auth-refresh-implementation.md on 2026-05-12 -->
   ```

## Tasks that span multiple `/keep-compile` runs

A single piece of work often takes multiple commits and multiple `/keep-compile` runs. The task stays `active` across all of them; `/keep-compile` updates the durable knowledge incrementally each time, and only marks the task `done` when the user signals (explicitly or by closing the work) that the task is complete.

When work is genuinely complete:

- The user can say "this task is done" or similar.
- Or `/keep-compile` can ask: *"Is this task complete? It's been associated with this work for a while and the latest knowledge updates look comprehensive."*
- On confirmation, `status: done` and archival happen.

If the user is unsure, leave the task `active`. Premature `done` marks cause confusion later when work continues.

## Re-opening archived tasks

If work resumes on something already archived (because it was prematurely closed, or because a follow-up emerged):

1. **Don't un-archive in place.** Create a fresh task with a new filename, possibly referencing the archived one in the body.
2. **Or move the archived file back to `/knowledge/tasks/`** and set `status: active`. The `created` date stays the original.

The first option is usually cleaner — the archived task remains a record of what was, the new task captures what now is.

## What `/keep-govern` checks for tasks

When `/keep-govern` runs, it reports task issues separately from durable-knowledge issues:

```
Tasks needing attention:

Active for too long (>30 days, possibly stale):
- knowledge/tasks/auth-refresh-implementation.md  (active, created 2026-03-15)

Archived for cleanup consideration (>90 days):
- knowledge/tasks/_archive/2025-12-10-old-migration.md

Topic doesn't match any domain in INDEX:
- knowledge/tasks/new-thing.md  (topic: gizmo, but no "gizmo" domain exists)
```

For each issue, `/keep-govern` only suggests action — never modifies. The user decides whether to update statuses, dig out content for promotion, or clean up.

## Hard rules (full list)

- **Never auto-promote.** Always surface candidates and wait for explicit approval.
- **Never let task content become the sole record of a durable fact.** If something is true and important, it lives in `/knowledge/docs`. The task may also contain it (e.g. as work-in-progress notes), but the durable layer is the source of truth.
- **Tasks are never indexed in `INDEX.md`.** The index reflects durable knowledge only.
- **The frontmatter is managed by `/keep-compile` and `/keep-govern`.** The user can edit it manually but should not need to — and if they do, the canonical values are what KEEP writes next.
- **Status transitions follow the allowed graph.** No direct `done ↔ abandoned`.
- **Archived tasks are never deleted by KEEP automatically.** `/keep-govern` can suggest cleanup, but the user approves.
- **Promotion copies, it doesn't move.** Source task content is preserved.
- **Tasks span multiple `/keep-compile` runs naturally.** `status: active` is the default and stays that way until the user signals completion.
