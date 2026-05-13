---
name: keep
description: "Living engineering knowledge for a code repo — specs, ADRs, architecture, runbooks — kept coherent as code evolves. Use on the /keep-init, /keep-retrieve, /keep-observe, /keep-compile, /keep-govern commands, on plain-language equivalents ('what did we decide about X', 'document this change', 'load context for Y', 'should we write an ADR'), and proactively after non-trivial changes in any repo that has a /knowledge directory."
---

# KEEP — Knowledge Engine for Engineering Persistence

A lightweight, memory-centric layer that maintains *living engineering knowledge* alongside a code repository. KEEP exists because code changes constantly while knowledge usually does not — and the gap between the two is where systems become incomprehensible.

KEEP treats the repository as a cognitive system:

- **code** = execution layer
- **`/knowledge`** = memory layer
- **the agent (you)** = reasoning layer

The skill is successful when a new agent or engineer can understand *how and why* the system evolved without re-inferring architecture from code alone.

---

## When to engage

**Engage immediately** in any of these situations — do not wait for explicit confirmation:

1. The user invokes one of the commands (`/keep-init`, `/keep-retrieve`, `/keep-observe`, `/keep-compile`, `/keep-govern`) or names KEEP or RepoMemory.
2. The user asks something equivalent in plain language. Examples that should trigger this skill even though they never say "KEEP":
   - *"What did we decide about the auth flow?"*
   - *"Load the context for the inference pipeline"*
   - *"Should I document this change somewhere?"*
   - *"Where do notes about this go?"*
   - *"Track this decision"* / *"Capture the reasoning"*
   - *"What's the current architecture of X"* in a repo that has knowledge files
3. **Proactive trigger** — the repository contains a `/knowledge` directory AND the user is making or has just made a non-trivial change. Categories that almost always warrant engagement:
   - Adding a new feature or significantly extending an existing one
   - Swapping a dependency, framework, or runtime
   - Changing system topology (new service, new boundary, new flow)
   - Fixing a bug following an incident (runbook candidate)
   - Modifying configuration that changes runtime behavior

   When you spot one of these in a `/knowledge`-enabled repo, do not silently complete the code work. Offer to run `/keep-observe` and propose updates. This is the entire reason the skill exists.

4. **First-time adoption** — the repository has no `/knowledge` directory but obviously has documentation lying around (a `docs/` folder, `ARCHITECTURE.md`, `RUNBOOK.md`, an `adr/` directory, scattered design notes) AND the user is asking about decisions, intent, or where things should go. Offer `/keep-init` rather than answering ad hoc.

**Do not engage** for:

- Trivial edits (typos, formatting, single-line fixes with no semantic impact).
- Repositories without a `/knowledge` folder, unless the user is asking to set one up or condition 4 above applies.
- Bulk retroactive documentation of the existing codebase — KEEP is brownfield-first and grows incrementally.

When in doubt, lean toward engaging and asking a quick check question rather than skipping. Under-triggering is the more common failure mode.

---

## Core principles (apply to every action)

1. **Memory-centric, not documentation-centric.** Capture rationale, architecture, constraints, and operational learnings. Skip implementation walkthroughs and prose that just re-narrates the code.

2. **Brownfield-first.** Work incrementally on existing repos. Never require a full upfront documentation pass. Document only the areas touched by the current change — or migrate from existing docs via `/keep-init` + `/keep-compile`.

3. **Conservative knowledge compilation.** Never invent architecture or intent. Stick to what the diff and the existing docs actually support. Good: *"Added JWT refresh flow."* Bad: *"The system now follows zero-trust distributed authentication."*

4. **Minimal diffs.** When updating existing knowledge files, change as little as possible. Preserve human-written rationale verbatim. Do not rewrite whole files.

5. **Durable vs ephemeral separation.** Specs, decisions, architecture, runbooks are durable — they live in `/knowledge/docs`. Tasks and scratchpads are ephemeral — they live in `/knowledge/tasks` and can be summarized, archived, or deleted. Never mix the two.

6. **Precision over completeness in retrieval.** When loading context, prefer small relevant slices over giant dumps. Token usage is a real cost; irrelevant context degrades reasoning.

7. **Ask before writing low-confidence content.** A diff shows *what* changed; it rarely shows *why*, what alternatives were rejected, what edge cases the author had in mind, or what failure modes the runbook should anticipate. That knowledge lives in the user's head. When `/keep-compile` is about to write a high-stakes field (rejected alternatives in an ADR, edge cases in a new spec, the cause field of a runbook) and the diff doesn't unambiguously settle it, stop and ask. See the **Eliciting tacit knowledge** section below for the protocol — including when *not* to ask, so the workflow does not become an interrogation.

---

## Repository structure

```
/knowledge
├── docs/
│   ├── specs/          — what the system should do (living specs)
│   ├── decisions/      — why the system is the way it is (ADRs)
│   ├── architecture/   — system topology, components, flows
│   └── runbooks/       — operational knowledge, failure modes, fixes
│
├── tasks/              — ephemeral execution state (NOT durable memory)
│
└── INDEX.md            — auto-generated navigation map (updated only by /keep-compile)
```

`/keep-init` scaffolds this layout. Empty subdirectories and a stub `INDEX.md` are fine — do **not** retroactively document the whole codebase.

For the precise format and conventions of each file type (spec template, ADR template, runbook template, architecture style), read `references/file_formats.md` when needed.

---

## Monorepo support

KEEP uses **a single `/knowledge/` directory at the repository root**, even in monorepos. There are no per-package knowledge directories.

The convention inside the single zone:

- `specs/` and `runbooks/` get a **per-package subdirectory** (`specs/auth-service/`, `runbooks/inference-api/`, `specs/shared/` for cross-cutting). `/keep-observe` infers the affected package from file paths in the diff; `/keep-compile` writes spec and runbook updates under the matching subdirectory.
- `decisions/` is **flat**. ADRs are numbered sequentially across the whole monorepo. The decision title and **Related** links indicate scope.
- `architecture/` is **flat**. One file per package (`auth-service.md`, `inference-api.md`) plus an `overview.md` for cross-cutting topology.
- `INDEX.md` adds a `## Packages` section at the top listing each package's spec/runbook/architecture entry points.

In a single-package repo this collapses to the simpler `specs/<area>/` convention with no `## Packages` section. All commands behave identically; only the file layout differs.

For full details — the rationale, the cross-package decision tree, and worked monorepo examples — read `references/monorepo.md`.

---

## The five commands

KEEP intentionally exposes a small, fixed set of commands. Each has a narrow contract — keep them separate; do not let one bleed into another.

### `/keep-init`

**Purpose:** bootstrap KEEP in a repo. Scaffolds `/knowledge`, detects monorepo layout, scans for pre-existing documentation, and produces an ingestion proposal. Does **not** migrate.

**Behavior:**
- If `/knowledge` already exists, do not overwrite. Report state and confirm.
- Detect monorepo shape (`pnpm-workspace.yaml`, package.json workspaces, Cargo workspace, `go.work`, top-level `apps/` `packages/` `services/`) and ask the user to confirm the package list before scaffolding per-package subdirs.
- Create the skeleton (`docs/{specs,decisions,architecture,runbooks}`, `tasks/`, stub `INDEX.md`).
- Run the pre-existing doc scan described in **Pre-existing documentation auto-detection** below. Output an ingestion proposal in the same format as `/keep-observe` ingestion mode.
- Append the KEEP workflow snippet to `CLAUDE.md` / `AGENTS.md` / `.cursorrules` (see `references/setup.md`).

**Hard rules:**
- Never overwrite an existing `/knowledge` directory without explicit confirmation.
- No migration during init. Files are *proposed* only; `/keep-compile` is the one that writes ingested content with per-file approval.

### `/keep-retrieve <topic>`

**Purpose:** load small, focused context relevant to a task.

**Behavior:**
- Identify which knowledge files touch the topic (use `INDEX.md` as the primary map, then fall back to filename and content search if needed).
- Surface only the relevant files; do not dump the whole `/knowledge` tree.
- Return a short list grouped by category (Architecture / Decisions / Specs / Runbooks). Each item is just a path; the user can ask to open any of them.

**Hard rules:**
- No file writes. `/keep-retrieve` is read-only.
- No speculation about what *might* be relevant from the code. Stick to what's indexed.
- If `INDEX.md` is empty or missing, say so and suggest running `/keep-init` (first time) or `/keep-observe` + `/keep-compile` (to start populating it).

### `/keep-observe [source]`

**Purpose:** semantic analysis of changes, classifying their impact on the knowledge layer. Suggests updates — does **not** apply them.

**Sources accepted.** `/keep-observe` works with any of the following, in this order of preference:

1. **A specific source the user names** — a branch (`/keep-observe feature/auth-refresh`), a PR (`/keep-observe PR#142` or a URL), a tag (`/keep-observe v2.3.0`), a commit range (`/keep-observe main..feature/x`), or a folder of existing documentation to ingest (`/keep-observe ./old-docs/`).
2. **The working diff** — `git diff` against the branch's merge base, or against `main` / `master` / the default branch.
3. **The last commit** — `git diff HEAD~1` as final fallback.

When a PR or branch is provided, also extract the **commit list with messages** — commit messages frequently carry the rationale that the diff alone hides ("switched from KServe because of CRD complexity"). When a tag is provided, compare against the previous tag.

**Combined git + docs signal.** Even when the source is a git artifact, scan for pre-existing docs under or near the touched paths (e.g. a diff in `services/auth/` should also pull `services/auth/README.md`, `services/auth/docs/*`, `docs/auth*`). Surface them as ingestion candidates *alongside* the diff classification — they often carry rationale the commits don't and should be folded into the same proposal.

**Pure-doc-folder mode.** When the source is a folder (`/keep-observe ./old-docs/`), shift into ingestion classification — classify each doc by likely target type, propose target paths, flag mixed content for split. See `references/brownfield.md` for the full heuristic catalog.

**Behavior:**
- Gather changes from the source.
- Categorize into: **Feature** (new or modified behavior), **Architecture** (structural/topological change), **Decision** (a choice with rejected alternatives that should be recorded), **Operational** (something a runbook should capture).
- For each category, list the affected knowledge files and what kind of update they likely need.
- Surface relevant **commit messages** that contain rationale-worthy phrases ("because", "instead of", "we tried", "this breaks", "incident", "rejected"). These are gold for ADR alternatives and runbook causes.
- If a `CHANGELOG.md` exists in the repo, scan its unreleased section as a secondary signal — but the PR commit history is the primary source when both are available.
- Output the classification only. No file modifications.

**Hard rules:**
- Never modify any file under `/knowledge`.
- Be conservative: if a change is purely refactoring with no semantic impact, say so and suggest no updates.
- If the source is empty, ambiguous, or unavailable, ask the user to clarify — do not guess.
- Quote commit messages or doc fragments rather than paraphrasing them when they contain rationale signals — they are evidence the user can verify.

See `references/observe_examples.md` for worked examples covering diffs, PRs, tags, and doc folders.

### `/keep-compile`

**Purpose:** apply the suggested updates from `/keep-observe` (or the ingestion proposal from `/keep-init`) to the durable knowledge layer.

**Behavior:**
- Take the previous `/keep-observe` (or `/keep-init`) output as input. If neither was run in this session, run `/keep-observe` first.
- For each suggested update:
  - **Updating an existing file:** make the smallest possible diff. Preserve all human-written content and rationale. Append or amend; do not rewrite.
  - **Creating a new ADR:** only when the change involves a non-trivial choice with rejected alternatives. Number it sequentially (ADR-NNNN-short-slug.md). Use the template in `references/file_formats.md`. **If this ADR supersedes a previous one**, see the supersession protocol in `references/file_formats.md` — both files must be updated and cross-linked.
  - **Updating specs:** add new requirements, edge cases, or acceptance criteria. Leave existing ones untouched unless the behavior actually changed.
  - **Updating runbooks:** add new failure modes and mitigations based on real operational evidence.
  - **Updating architecture:** only when topology changed.
  - **Migrating a pre-existing doc (ingestion):** map source content into the target template, **quote source content verbatim** (no paraphrasing — restructure / dedup / typo fixes only), run elicitation for missing high-stakes fields, add a provenance comment `<!-- Migrated from <source> on YYYY-MM-DD -->`. Per-file approval — never migrate the folder in one shot. See `references/brownfield.md`.
- **Add cross-references** when relevant: a spec that implements an ADR should link to it; an ADR that affects an architecture component should be linked from there. See the linking convention in `references/file_formats.md`.
- **Apply the elicitation protocol** before writing high-stakes fields. For ADRs and new specs, run elicitation in **batch mode** (collect all answers first, then write); for runbooks and incremental updates use reactive mode. See `references/elicitation.md`.
- After all knowledge file updates, **incrementally update `INDEX.md`**: only regenerate the sections affected by the changes (domains touched, new entities introduced, flows that crossed boundaries). Preserve the rest of `INDEX.md` byte-for-byte. The first-ever `/keep-compile` on a repo regenerates `INDEX.md` from scratch.
- **Handle the `/knowledge/tasks` lifecycle**: see the task lifecycle section below.
- Print a summary: what was created, what was updated, which task files were promoted or archived, and any updates you declined to make (with the reason).

**Hard rules:**
- Minimal diffs only. If you find yourself rewriting a paragraph, stop and reconsider.
- No duplication. Before adding a new section, check whether it already exists elsewhere.
- Never delete existing rationale, even if it looks outdated — mark it as superseded instead.
- If `/keep-observe` was not run in this session, run it first.
- **Before writing high-stakes fields**, apply the elicitation protocol. Do not invent rejected alternatives, edge cases, or root causes from the diff alone.
- **Never auto-migrate task content** into durable docs without surfacing it to the user first.
- **Per-file approval for ingestion.** No batch migration of doc folders.
- **Source files are never modified.** Pre-existing docs are read-only — KEEP reads, never writes.
- **Verify filesystem state before claiming any sequential or set-based fact.** Before assigning an ADR number, `ls` the `decisions/` directory and use the next available integer — never pre-number from memory or from the conversation flow. The same applies before claiming a domain doesn't exist, an entity isn't in the INDEX, or a file is missing. Check, don't assume.

### `/keep-govern` *(periodic)*

**Purpose:** detect knowledge entropy — stale docs, duplication, contradictions, oversized files. Suggests cleanup; never auto-deletes.

**Behavior:**
- Scan `/knowledge/docs` for:
  - Files not touched in a long time (>6 months heuristic) where the corresponding code area has changed.
  - Pairs of files whose titles or content overlap heavily.
  - ADRs that contradict each other without an explicit supersedes link.
  - Files that have grown beyond ~300 lines and should be split.
  - **Files with `<!-- TODO(KEEP): ... -->` markers** — these signal incomplete knowledge from prior compiles where the user declined to answer elicitation questions. Surface them so they can be enriched now that context may be fresher.
  - **INDEX.md with more than ~5 single-entry domains** — suggests the domain granularity is too fine; consider collapsing.
- Scan `/knowledge/tasks` for **active tasks older than 30 days** and **archived tasks older than 90 days** (candidates for `_archive/` cleanup, never deleted).
- Output suggestions grouped by action: *archive*, *merge*, *summarize*, *split*, *enrich*, *collapse*. Each suggestion explains why.
- Wait for explicit user approval before applying any of the suggestions.

**Hard rules:**
- Suggestions only. Never auto-delete or auto-merge.
- Preserve historical rationale. If archiving, move to `/knowledge/docs/_archive/` rather than deleting.

---

## Pre-existing documentation auto-detection

KEEP treats pre-existing markdown documentation as a first-class input source — equal in importance to git history. Two commands use this:

- **`/keep-init`** scans the whole repo and produces the initial ingestion proposal.
- **`/keep-observe`** (without a source argument, or with a git source) opportunistically scans paths near the diff and folds candidates into the same proposal.

### Locations to scan

These are the canonical places where pre-existing docs tend to live. Walk them recursively, excluding `node_modules`, `.git`, `dist`, `build`, `target`, `vendor`, `.venv`, and any path already under `/knowledge`:

- `README.md` at the repo root and inside any package
- `docs/`, `doc/`, `documentation/` at any depth
- `ARCHITECTURE.md`, `ARCHITECTURE/`, `DESIGN.md`, `DESIGN/`
- `RUNBOOK.md`, `RUNBOOKS/`, `runbook/`, `runbooks/`
- `notes/`, `design-notes/`, `rfc/`, `rfcs/`, `adr/`, `decisions/`
- `wiki/`, `.wiki/`
- Top-level `*.md` other than typical project boilerplate (LICENSE, CONTRIBUTING, CODE_OF_CONDUCT, CHANGELOG)

### Classification heuristics

Classify each file by combining **heading-pattern signals** (structural) with **keyword signals** (semantic). When heading and keyword signals disagree, prefer the heading signal — structure is more reliable than wording. When neither signal is conclusive, flag as **unclear** and ask the user.

**Likely spec** — heading pattern includes things like `## Endpoint`, `## API`, `## Behavior`, `## Requirements`, `## Acceptance criteria`, `## Inputs / Outputs`, `## Validation`, `## Edge cases`. Keywords: "shall", "must return", "is expected to", "given … when … then", "rejects", "accepts".

**Likely runbook** — heading pattern includes `## Symptoms`, `## Detection`, `## Cause`, `## Mitigation`, `## Resolution`, `## Rollback`, `## Postmortem`, `## Recovery`. Keywords: "alert", "page", "on-call", "incident", "post-mortem", "RTO", "RPO", "rollback", "5xx", "p99", "OOM".

**Likely ADR / decision** — heading pattern includes `## Context`, `## Decision`, `## Status`, `## Alternatives`, `## Consequences`, `## Trade-offs`. Keywords: "because", "instead of", "we chose", "we rejected", "we tried … but", "supersedes", "deprecates", "the trade-off is".

**Likely architecture** — heading pattern includes `## Topology`, `## Components`, `## Flows`, `## Boundaries`, `## System overview`, `## Dependencies`, `## Data flow`. Keywords: "service mesh", "boundary", "owns", "depends on", "produces", "consumes", topology diagrams.

**Mixed content** — the file matches multiple categories with substantive content for each (e.g. meeting notes that include a decision rationale + a runbook entry + scratch). Propose a split rather than guessing.

**Discard candidates** — pure code snippets, dependency lists, build instructions, user-facing how-tos, todo lists. These are not durable knowledge; flag and let the user decide.

### Target-path inference

After classification, propose a target path:

- **specs:** `/knowledge/docs/specs/<package-or-area>/<slug>.md`. Infer the package from the source path (`services/auth/docs/jwt.md` → `specs/auth/jwt.md`). In single-package repos, use `<area>` derived from filename or top heading.
- **runbooks:** `/knowledge/docs/runbooks/<package-or-area>/<slug>.md` or `runbooks/shared/<slug>.md` for cross-cutting.
- **ADRs:** `/knowledge/docs/decisions/ADR-NNNN-<slug>.md` where NNNN is the next available number (compute at compile time, not at observe — `/keep-observe` proposes `ADR-NNNN` as a placeholder).
- **architecture:** `/knowledge/docs/architecture/<area>.md` (one file per package, plus `overview.md` for system-wide).

When the source path strongly suggests a package (`packages/foo/docs/...`), use that package; otherwise ask the user.

### Output format

The classification output is the same shape as `/keep-observe` in ingestion mode (see `references/brownfield.md` for a worked example). Each candidate carries:

- source path
- proposed target type and path
- one-line rationale ("matches the runbook template — has symptoms/cause/mitigation headings")
- elicitation flags ("source lists alternatives but no rejection rationale — will need batch interview at compile time")

Mixed-content files are presented with a proposed split; unclear files are listed separately for user input. **Nothing is written.** Migration only happens through `/keep-compile`, file-by-file, with explicit approval.

---

## Task lifecycle

`/knowledge/tasks` exists for ephemeral execution state — implementation plans, scratch notes, debug traces. Each task file carries a minimal YAML frontmatter that **`/keep-compile` and `/keep-govern` manage automatically** — the user is not expected to maintain it by hand:

```yaml
---
status: active        # active | done | abandoned
created: 2026-05-12
topic: auth-refresh
---
```

The lifecycle in brief:

- **Creation** — task files are written freely during work. If created without frontmatter, `/keep-compile` inserts a default block on first touch.
- **Promotion** — during `/keep-compile`, if a task contains content that looks like durable knowledge (rationale, edge case, observed failure), KEEP surfaces it as a promotion candidate. Migration happens only with user approval and the source task is preserved.
- **Status transition** — when work concludes, `/keep-compile` updates `status` to `done` (work reflected in durable knowledge) or `abandoned` (work dropped).
- **Archival** — `done` and `abandoned` tasks move to `/knowledge/tasks/_archive/<YYYY-MM-DD>-<name>.md` where the date comes from `created`. `active` tasks are never archived.
- **Governance** — `/keep-govern` surfaces `active` tasks older than 30 days, archived tasks older than 90 days, and tasks whose `topic` doesn't match any domain.

**Hard rules:** never auto-promote without user approval; never let task content become the sole record of a durable fact; tasks are never indexed in `INDEX.md`; the frontmatter is managed by KEEP, not the user.

For full operational detail, including how `/keep-compile` decides what to promote and how to handle tasks that span multiple compile runs, read `references/tasks.md`.

---

## Eliciting tacit knowledge

A diff is a poor source for *intent*. It shows the new code, not the alternatives the author considered and rejected, not the edge cases they had in mind, not the production incident behind a runbook entry. That information lives in the user's head. KEEP captures it by *asking*, not by inferring.

This protocol applies primarily to `/keep-compile`, the only command that writes durable content. It is a principle, not a separate command — there is deliberately no `/elicit` step the user can skip.

### When to ask

Ask before writing if **all three** are true:

1. The field is **high-stakes** — its value carries information the diff alone cannot establish.
2. You have **low confidence** — what the diff shows could be explained multiple ways, or critical context is missing.
3. The field is being **created or substantially extended**, not just touched.

High-stakes fields, by file type:

- **ADR — rejected alternatives, consequences.** A decision without rejected alternatives is not a decision; without consequences, it is not actionable. Never fabricate either.
- **Spec — edge cases, acceptance criteria.** Code shows the happy path; specs must capture the cases the code handles deliberately and the criteria that prove it.
- **Runbook — symptoms, causes.** Speculative runbooks are worse than no runbook. Only write what was actually observed in production.
- **Architecture — boundaries, constraints.** What is intentionally outside the component, and what invariant must hold.

### When not to ask

Do **not** ask if any of the following:

- You are updating an existing field with content the diff clearly supports (e.g. adding one requirement to an existing spec, where the diff plainly implements that requirement).
- **The diff or its commit messages contain explicit evidence** for the high-stakes field. If a commit message says *"chose Ray Serve over KServe because of Python ergonomics; KServe's CRDs added too much complexity"*, the rejected alternative and its rationale are already on record — quote them into the ADR rather than re-asking. The user explicitly captured this knowledge; making them re-state it is friction without value. Quote the evidence verbatim in the file (with the commit SHA in a comment if helpful) so the source is traceable.
- The user has just answered a question on the same topic in this session.
- The change is purely operational (formatting, renaming) with no semantic content to capture.
- You can write a conservative, factually-grounded entry without speculating — and the missing detail is *enrichment*, not load-bearing.

The rule of thumb: **ask only for what the diff, commits, and conversation cannot already establish.** Elicitation is for tacit knowledge — knowledge that lives only in the user's head. Anything they already wrote down somewhere is not tacit anymore.

### How to ask

Four rules that keep elicitation from becoming an interrogation:

1. **Choose batch vs reactive mode based on file type.**
   - **Batch mode** for **new ADRs and new specs.** The load-bearing fields are correlated — rejected alternatives depend on context, consequences depend on the decision, edge cases depend on the requirements scope. Asking them one at a time wastes turns and produces inconsistent answers. Collect all the questions up front in a single turn (using `ask_user_input_v0` with multiple questions where possible), then write the file in one pass.
   - **Reactive mode** for **runbooks and incremental updates to existing files.** Runbook fields are mostly independent. Incremental updates touch one thing at a time. Ask only what is needed for that specific update.

2. **Cap at three questions per turn.** Whether batch or reactive. If a batch interview would require more than three questions, prioritize the load-bearing ones and flag the rest as follow-up for after the file is drafted.

3. **Use structured inputs when the environment offers them.** On Claude.ai with the `ask_user_input_v0` tool available, present discrete options as tappable choices (single-select for one-of-N, multi-select for "which of these apply", `rank_priorities` for ordering trade-offs). Free-text only for genuinely open questions. This dramatically reduces friction. In terminal-only environments (Claude Code, Cursor without tappable UI), ask in plain text but still offer enumerated options the user can answer with a number or letter.

4. **Show your tentative draft alongside the question.** "Here's what I'd write based on the diff alone — does this miss anything?" is more useful than an abstract "tell me the edge cases". The user reacts to concrete content much faster than to open questions.

For the full taxonomy of which questions to ask for each file type, with good and bad examples and batch interview templates, read `references/elicitation.md`.

### What to do with the answers

Answers go **directly into the durable knowledge file** being written. Never into `/knowledge/tasks` as intermediate state — that creates a third place where knowledge lives and breaks the durable/ephemeral separation.

**Partial answers in batch mode.** When you ask three questions in batch and the user answers only some of them (skips one, says "I don't know", or leaves a field blank), do not re-ask the unanswered questions in a follow-up turn. Treat each unanswered field individually:

- If the field is **load-bearing** (rejected alternatives in an ADR, edge cases in a new spec) and unanswered, omit the section entirely and insert a TODO marker (see below).
- If the field is **enrichment** (e.g. "what should we revisit later"), simply omit it from the file. No marker needed — its absence is not a problem.

Write the file once with what you have. Pushing for the missing answers in a second turn is the interrogation pattern this protocol is designed to prevent.

**When the user declines entirely** ("just write your best guess", "skip the questions"), write a conservative version that omits the uncertain fields entirely rather than fabricating them. An ADR with no `Alternatives considered` section is better than one with invented alternatives. Mark such files with a TODO marker for future enrichment:

```md
<!-- TODO(KEEP): rejected alternatives not captured at compile time -->
```

This makes them findable later by `/keep-govern` and signals honestly that the file is incomplete.

---

## The standard workflow

```
First time on a repo:
  /keep-init                       ← scaffold /knowledge + scan pre-existing docs
  /keep-compile                    ← migrate ingestion candidates one by one (with elicitation)

Ongoing:
  /keep-retrieve <topic>           ← before starting work, load focused context
  [implement code changes]
  /keep-observe                    ← after changes, classify impact (git diff + nearby docs)
  /keep-compile                    ← apply the suggested knowledge updates
  /keep-govern                     ← periodically, not every cycle
```

When triggered proactively (user is making changes in a repo with `/knowledge` but hasn't invoked KEEP), suggest the workflow rather than silently running it:

> "I notice this repo has a `/knowledge` directory and the change you just made touches the auth flow. Would you like me to run `/keep-observe` and propose knowledge updates?"

For first-time adoption on a repo that already has docs lying around but no `/knowledge`:

> "This repo has a `docs/` folder and an `ARCHITECTURE.md` but no `/knowledge`. Want me to run `/keep-init`? It'll scaffold the layout and propose how to migrate the existing docs — no writes until you approve them one by one."

Wait for confirmation before running `/keep-compile`. The user owns the knowledge layer; the skill is an aid, not an autopilot.

---

## Output formats

### `/keep-retrieve` output template

```
Relevant context for "<topic>":

Architecture:
- knowledge/docs/architecture/<file>.md

Decisions:
- knowledge/docs/decisions/ADR-NNNN-<slug>.md

Specs:
- knowledge/docs/specs/<area>/<file>.md

Runbooks:
- knowledge/docs/runbooks/<file>.md
```

Omit sections that have no matches. If nothing matches, say so plainly: *"No indexed knowledge for `<topic>` yet. Run `/keep-observe` after making changes to start building it."*

### `/keep-observe` output template

```
Detected changes:

Feature:
- <one-line description of the behavioral change>

Architecture:
- <one-line description of the structural change>

Operational:
- <one-line description of the operational learning>

Decision:
- <one-line description, if a real choice was made>

Suggested knowledge updates:
- knowledge/docs/specs/<area>/<file>.md  (add: <what>)
- knowledge/docs/decisions/ADR-NNNN-<slug>.md  (create)
- knowledge/docs/runbooks/<file>.md  (add: <what>)

Ingestion candidates from nearby docs:
- ./services/auth/README.md → knowledge/docs/specs/auth/overview.md  (likely spec — matches API/Behavior headings)
- ./docs/auth-flow.md → knowledge/docs/architecture/auth.md  (likely architecture — describes topology)
```

Omit empty categories. Be specific about *what* the update should add, not just *which* file. The ingestion section appears only when nearby pre-existing docs were detected.

### `/keep-compile` output template

```
Created:
- knowledge/docs/decisions/ADR-NNNN-<slug>.md

Updated:
- knowledge/docs/specs/<area>/<file>.md
- knowledge/docs/runbooks/<file>.md

Migrated (from ingestion):
- knowledge/docs/specs/auth/overview.md  (from ./services/auth/README.md)

Updated INDEX.md

Skipped:
- <file>  (reason: <why no update was needed>)
```

---

## Anti-goals

KEEP must not become:

- a documentation generator that produces prose on demand
- a project management or ticketing system
- a heavyweight SDD framework with upfront design ceremonies
- an AI wiki that drowns the repo in auto-generated text
- a workflow engine with lifecycle states and gates

If a request would push KEEP in any of these directions, push back. The skill's value comes from staying small.

---

## Reference files

- `references/file_formats.md` — templates and conventions for specs, ADRs, runbooks, architecture docs, `INDEX.md`, and task frontmatter. Includes the cross-cutting **linking convention** and **ADR supersession** protocol. Read when creating or updating any knowledge file.
- `references/observe_examples.md` — worked examples of `/keep-observe` classifications covering diffs, PRs, tags, and doc folder ingestion. Read when the current source is ambiguous.
- `references/elicitation.md` — taxonomy of questions to ask for each file type, the evidence-first rule (when *not* to ask), batch vs reactive mode, and how to handle partial answers. Read before asking questions during `/keep-compile`.
- `references/monorepo.md` — full monorepo layout, command behaviors per-package, and the decision tree for cross-package content. Read when working in a monorepo or when the user is setting up KEEP on one.
- `references/brownfield.md` — operational detail for `/keep-init` scanning and `/keep-observe ./folder/` ingestion, including the full classification heuristic catalog and how to handle mixed-content source files. Read when adopting KEEP on an established project or importing knowledge from other agents.
- `references/tasks.md` — full task lifecycle, promotion mechanics, status transition rules, and `/keep-govern` behavior on tasks. Read when handling `/knowledge/tasks/` content during `/keep-compile`.
- `references/setup.md` — agent-instruction snippet for `CLAUDE.md` / `AGENTS.md` / `.cursorrules`, common adoption mistakes, and the verification loop. Read when running `/keep-init` or when the user is adopting KEEP for the first time.

---

## Core insight

Code stores execution logic.

KEEP stores everything around it — architecture, rationale, operational learnings, feature evolution — so that the repository remains *understandable* as it grows. The repository is not just source code; with KEEP, it is persistent engineering cognition.
