---
name: keep
description: "Maintain living engineering knowledge for a code repo — a memory-centric, lightweight SDD system that keeps architecture, specs, decisions (ADRs), and runbooks coherent as code evolves. ALWAYS use when the user mentions KEEP, RepoMemory, 'repository cognition', 'living specs', 'knowledge layer', ADRs, or invokes /retrieve, /observe, /compile, /govern. ALSO use whenever the user asks to 'update the knowledge base', 'document the change', 'load context for X', 'remember why we did Y', 'capture the reasoning', 'what did we decide about Z', 'track this decision', 'should we write an ADR' — even without naming KEEP. ALWAYS engage proactively when working in a repo that contains a /knowledge directory AND the user is making non-trivial code changes (new features, architectural shifts, dependency swaps, incident fixes). Covers the four commands, durable-vs-ephemeral knowledge, file linking, ADR supersession, batch-mode elicitation for high-stakes content, monorepo layout, and brownfield doc ingestion."
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

1. The user invokes one of the commands (`/retrieve`, `/observe`, `/compile`, `/govern`) or names KEEP or RepoMemory.
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

   When you spot one of these in a `/knowledge`-enabled repo, do not silently complete the code work. Offer to run `/observe` and propose updates. This is the entire reason the skill exists.

**Do not engage** for:

- Trivial edits (typos, formatting, single-line fixes with no semantic impact).
- Repositories without a `/knowledge` folder, unless the user is asking to set one up.
- Bulk retroactive documentation of the existing codebase — KEEP is brownfield-first and grows incrementally.

When in doubt, lean toward engaging and asking a quick check question rather than skipping. Under-triggering is the more common failure mode.

---

## Core principles (apply to every action)

1. **Memory-centric, not documentation-centric.** Capture rationale, architecture, constraints, and operational learnings. Skip implementation walkthroughs and prose that just re-narrates the code.

2. **Brownfield-first.** Work incrementally on existing repos. Never require a full upfront documentation pass. Document only the areas touched by the current change.

3. **Conservative knowledge compilation.** Never invent architecture or intent. Stick to what the diff and the existing docs actually support. Good: *"Added JWT refresh flow."* Bad: *"The system now follows zero-trust distributed authentication."*

4. **Minimal diffs.** When updating existing knowledge files, change as little as possible. Preserve human-written rationale verbatim. Do not rewrite whole files.

5. **Durable vs ephemeral separation.** Specs, decisions, architecture, runbooks are durable — they live in `/knowledge/docs`. Tasks and scratchpads are ephemeral — they live in `/knowledge/tasks` and can be summarized, archived, or deleted. Never mix the two.

6. **Precision over completeness in retrieval.** When loading context, prefer small relevant slices over giant dumps. Token usage is a real cost; irrelevant context degrades reasoning.

7. **Ask before writing low-confidence content.** A diff shows *what* changed; it rarely shows *why*, what alternatives were rejected, what edge cases the author had in mind, or what failure modes the runbook should anticipate. That knowledge lives in the user's head. When `/compile` is about to write a high-stakes field (rejected alternatives in an ADR, edge cases in a new spec, the cause field of a runbook) and the diff doesn't unambiguously settle it, stop and ask. See the **Eliciting tacit knowledge** section below for the protocol — including when *not* to ask, so the workflow does not become an interrogation.

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
└── INDEX.md            — auto-generated navigation map (updated only by /compile)
```

If the user is adopting KEEP for the first time on an existing repo, set up just this skeleton — empty subdirectories and a stub `INDEX.md` are fine. Do **not** retroactively document the whole codebase.

For the precise format and conventions of each file type (spec template, ADR template, runbook template, architecture style), read `references/file_formats.md` when needed.

---

## Monorepo support

KEEP uses **a single `/knowledge/` directory at the repository root**, even in monorepos. There are no per-package knowledge directories.

The convention inside the single zone:

- `specs/` and `runbooks/` get a **per-package subdirectory** (`specs/auth-service/`, `runbooks/inference-api/`, `specs/shared/` for cross-cutting). `/observe` infers the affected package from file paths in the diff; `/compile` writes spec and runbook updates under the matching subdirectory.
- `decisions/` is **flat**. ADRs are numbered sequentially across the whole monorepo. The decision title and **Related** links indicate scope.
- `architecture/` is **flat**. One file per package (`auth-service.md`, `inference-api.md`) plus an `overview.md` for cross-cutting topology.
- `INDEX.md` adds a `## Packages` section at the top listing each package's spec/runbook/architecture entry points.

In a single-package repo this collapses to the simpler `specs/<area>/` convention with no `## Packages` section. All commands behave identically; only the file layout differs.

For full details — the rationale, the cross-package decision tree, and worked monorepo examples — read `references/monorepo.md`.

---

## The four commands

KEEP intentionally exposes exactly four commands. Each has a narrow contract — keep them separate; do not let one bleed into another.

### `/retrieve <topic>`

**Purpose:** load small, focused context relevant to a task.

**Behavior:**
- Identify which knowledge files touch the topic (use `INDEX.md` as the primary map, then fall back to filename and content search if needed).
- Surface only the relevant files; do not dump the whole `/knowledge` tree.
- Return a short list grouped by category (Architecture / Decisions / Specs / Runbooks). Each item is just a path; the user can ask to open any of them.

**Hard rules:**
- No file writes. `/retrieve` is read-only.
- No speculation about what *might* be relevant from the code. Stick to what's indexed.
- If `INDEX.md` is empty or missing, say so and suggest running `/compile` first.

### `/observe [source]`

**Purpose:** semantic analysis of changes, classifying their impact on the knowledge layer. Suggests updates — does **not** apply them.

**Sources accepted.** `/observe` works with any of the following, in this order of preference:

1. **A specific source the user names** — a branch (`/observe feature/auth-refresh`), a PR (`/observe PR#142` or a URL), a tag (`/observe v2.3.0`), a commit range (`/observe main..feature/x`), or a folder of existing documentation to ingest (`/observe ./old-docs/`).
2. **The working diff** — `git diff` against the branch's merge base, or against `main` / `master` / the default branch.
3. **The last commit** — `git diff HEAD~1` as final fallback.

When a PR or branch is provided, also extract the **commit list with messages** — commit messages frequently carry the rationale that the diff alone hides ("switched from KServe because of CRD complexity"). When a tag is provided, compare against the previous tag.

When a folder of existing docs is provided, treat it as a knowledge ingestion task: classify each document by likely target type (spec / ADR / architecture / runbook), surface candidates, and let `/compile` migrate them — do **not** auto-migrate from `/observe`.

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

### `/compile`

**Purpose:** apply the suggested updates from `/observe` to the durable knowledge layer.

**Behavior:**
- Take the `/observe` output (either from the current session or re-run it) as the input.
- For each suggested update:
  - **Updating an existing file:** make the smallest possible diff. Preserve all human-written content and rationale. Append or amend; do not rewrite.
  - **Creating a new ADR:** only when the change involves a non-trivial choice with rejected alternatives. Number it sequentially (ADR-NNNN-short-slug.md). Use the template in `references/file_formats.md`. **If this ADR supersedes a previous one**, see the supersession protocol in `references/file_formats.md` — both files must be updated and cross-linked.
  - **Updating specs:** add new requirements, edge cases, or acceptance criteria. Leave existing ones untouched unless the behavior actually changed.
  - **Updating runbooks:** add new failure modes and mitigations based on real operational evidence.
  - **Updating architecture:** only when topology changed.
- **Add cross-references** when relevant: a spec that implements an ADR should link to it; an ADR that affects an architecture component should be linked from there. See the linking convention in `references/file_formats.md`.
- **Apply the elicitation protocol** before writing high-stakes fields. For ADRs and new specs, run elicitation in **batch mode** (collect all answers first, then write); for runbooks and incremental updates use reactive mode. See `references/elicitation.md`.
- After all knowledge file updates, **incrementally update `INDEX.md`**: only regenerate the sections affected by the changes (domains touched, new entities introduced, flows that crossed boundaries). Preserve the rest of `INDEX.md` byte-for-byte. The first-ever `/compile` on a repo regenerates `INDEX.md` from scratch.
- **Handle the `/knowledge/tasks` lifecycle**: see the task lifecycle section below.
- Print a summary: what was created, what was updated, which task files were promoted or archived, and any updates you declined to make (with the reason).

**Hard rules:**
- Minimal diffs only. If you find yourself rewriting a paragraph, stop and reconsider.
- No duplication. Before adding a new section, check whether it already exists elsewhere.
- Never delete existing rationale, even if it looks outdated — mark it as superseded instead.
- If `/observe` was not run in this session, run it first.
- **Before writing high-stakes fields**, apply the elicitation protocol. Do not invent rejected alternatives, edge cases, or root causes from the diff alone.
- **Never auto-migrate task content** into durable docs without surfacing it to the user first.
- **Verify filesystem state before claiming any sequential or set-based fact.** Before assigning an ADR number, `ls` the `decisions/` directory and use the next available integer — never pre-number from memory or from the conversation flow. The same applies before claiming a domain doesn't exist, an entity isn't in the INDEX, or a file is missing. Check, don't assume.

### `/govern` *(optional, periodic)*

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

## Task lifecycle

`/knowledge/tasks` exists for ephemeral execution state — implementation plans, scratch notes, debug traces. Each task file carries a minimal YAML frontmatter that **`/compile` and `/govern` manage automatically** — the user is not expected to maintain it by hand:

```yaml
---
status: active        # active | done | abandoned
created: 2026-05-12
topic: auth-refresh
---
```

The lifecycle in brief:

- **Creation** — task files are written freely during work. If created without frontmatter, `/compile` inserts a default block on first touch.
- **Promotion** — during `/compile`, if a task contains content that looks like durable knowledge (rationale, edge case, observed failure), KEEP surfaces it as a promotion candidate. Migration happens only with user approval and the source task is preserved.
- **Status transition** — when work concludes, `/compile` updates `status` to `done` (work reflected in durable knowledge) or `abandoned` (work dropped).
- **Archival** — `done` and `abandoned` tasks move to `/knowledge/tasks/_archive/<YYYY-MM-DD>-<name>.md` where the date comes from `created`. `active` tasks are never archived.
- **Governance** — `/govern` surfaces `active` tasks older than 30 days, archived tasks older than 90 days, and tasks whose `topic` doesn't match any domain.

**Hard rules:** never auto-promote without user approval; never let task content become the sole record of a durable fact; tasks are never indexed in `INDEX.md`; the frontmatter is managed by KEEP, not the user.

For full operational detail, including how `/compile` decides what to promote and how to handle tasks that span multiple `/compile` runs, read `references/tasks.md`.

---

## Eliciting tacit knowledge

A diff is a poor source for *intent*. It shows the new code, not the alternatives the author considered and rejected, not the edge cases they had in mind, not the production incident behind a runbook entry. That information lives in the user's head. KEEP captures it by *asking*, not by inferring.

This protocol applies primarily to `/compile`, the only command that writes durable content. It is a principle, not a separate command — there is deliberately no `/elicit` step the user can skip.

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

This makes them findable later by `/govern` and signals honestly that the file is incomplete.

---

## The standard workflow

```
/retrieve <topic>          ← before starting work, load focused context
[implement code changes]
/observe                   ← after changes, classify impact
/compile                   ← apply the suggested knowledge updates
/govern                    ← periodically, not every cycle
```

When triggered proactively (user is making changes in a repo with `/knowledge` but hasn't invoked KEEP), suggest the workflow rather than silently running it:

> "I notice this repo has a `/knowledge` directory and the change you just made touches the auth flow. Would you like me to run `/observe` and propose knowledge updates?"

Wait for confirmation before running `/compile`. The user owns the knowledge layer; the skill is an aid, not an autopilot.

---

## Brownfield ingestion workflow

When `/observe` is given a folder of existing docs (`/observe ./old-docs/`) instead of a diff, it shifts into **ingestion mode** — for adopting KEEP on an established project or importing knowledge from other agents.

The flow is always three steps, with `/observe` doing classification only and `/compile` doing migration with user-by-file approval:

1. **`/observe ./old-docs/`** — reads each file, proposes a target type (spec / ADR / architecture / runbook) and target path. Flags unclear files for user input. Writes nothing.
2. **User reviews** — corrects mappings, drops candidates, accepts the rest. Cheap correction point before any files are written.
3. **`/compile`** — migrates each accepted candidate one at a time. Maps the source content into the target template, runs elicitation for missing high-stakes fields, quotes source content verbatim rather than paraphrasing, adds a provenance comment (`<!-- Migrated from ./old-docs/auth-design.md on YYYY-MM-DD -->`), and updates `INDEX.md` incrementally.

**Hard rules:** per-file approval (no en-masse migration), no paraphrasing of source content (the user's wording is evidence), elicitation still applies for missing load-bearing fields, provenance comments always, source folder never deleted by KEEP.

For the full workflow, including detailed examples and edge cases, read `references/brownfield.md`.

---

## Output formats

### `/retrieve` output template

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

Omit sections that have no matches. If nothing matches, say so plainly: *"No indexed knowledge for `<topic>` yet. Run `/observe` after making changes to start building it."*

### `/observe` output template

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
```

Omit empty categories. Be specific about *what* the update should add, not just *which* file.

### `/compile` output template

```
Created:
- knowledge/docs/decisions/ADR-NNNN-<slug>.md

Updated:
- knowledge/docs/specs/<area>/<file>.md
- knowledge/docs/runbooks/<file>.md

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
- `references/observe_examples.md` — worked examples of `/observe` classifications covering diffs, PRs, tags, and doc folder ingestion. Read when the current source is ambiguous.
- `references/elicitation.md` — taxonomy of questions to ask for each file type, the evidence-first rule (when *not* to ask), batch vs reactive mode, and how to handle partial answers. Read before asking questions during `/compile`.
- `references/monorepo.md` — full monorepo layout, command behaviors per-package, and the decision tree for cross-package content. Read when working in a monorepo or when the user is setting up KEEP on one.
- `references/brownfield.md` — operational detail for ingesting existing doc folders (`/observe ./folder/`), including how to handle mixed-content source files. Read when adopting KEEP on an established project or importing knowledge from other agents.
- `references/tasks.md` — full task lifecycle, promotion mechanics, status transition rules, and `/govern` behavior on tasks. Read when handling `/knowledge/tasks/` content during `/compile`.
- `references/setup.md` — how to initialize KEEP in a brownfield repo and what to add to `CLAUDE.md` / `AGENTS.md` / `.cursorrules`. Read when the user is adopting KEEP for the first time.

---

## Core insight

Code stores execution logic.

KEEP stores everything around it — architecture, rationale, operational learnings, feature evolution — so that the repository remains *understandable* as it grows. The repository is not just source code; with KEEP, it is persistent engineering cognition.
