---
name: keep
description: "Living /knowledge layer for a code repo — specs, ADRs, ideas, with runbook and architecture as tags, and anchored facts that drift-detection verifies deterministically. CONSULT whenever the user asks how the system works, why a decision was made, what conventions exist, or whether a change is safe — even without naming KEEP. ALSO trigger on non-trivial changes that affect behavior/architecture/operations in a /knowledge-enabled repo, brownfield doc cordoning, parked ideas ('not now but', 'what if we'), pre-merge drift checks, and questions about anchor coverage. SKIP for generic programming Q&A, library docs, pure mechanical refactors, and repos without /knowledge."
---

# KEEP — Knowledge Engine for Engineering Persistence

KEEP turns a code repository into a **living knowledge graph** that an LLM agent can both consult and maintain. It is the answer to the question Karpathy posed for personal wikis applied to software: *what if every project had a wiki the LLM kept up to date for us, that we could query reliably?*

KEEP treats the repository as a cognitive system:

- **code** = execution layer
- **`/knowledge`** = memory layer
- **agent (you)** = reasoning layer

The skill is successful when a new agent or engineer can answer *how* and *why* the system evolved without re-deriving everything from code, and when changes to code automatically surface as proposed changes to knowledge.

---

## How to trigger this skill (read this section closely)

KEEP is failing if it triggers only on code changes. Code is half the loop. The **read path** — consulting knowledge when answering questions — is the half that compounds value.

### Strong read triggers — ALWAYS run `/keep-ask` first

If the user's prompt contains any of these patterns, `/keep-ask` is the correct first action before answering:

- *"how does X work?"*
- *"why did we choose Y?"* / *"why are we doing Y this way?"*
- *"what did we decide about Z?"* / *"did we ever decide on Z?"*
- *"is there a convention for"* / *"how do we usually do"*
- *"where do I look for"* / *"where does X live"*
- *"what happens when"* / *"what's the behavior if"*
- *"are we using X or Y?"* / *"what's our stack for"*
- *"is there already something for"* / *"do we have a"*
- *"is it safe to change"* / *"what depends on this"*

The pattern: the user is reaching for information that *should be* in `/knowledge`. Even if the answer might also be in the code, the knowledge layer has the **why** (rationale, rejected alternatives, edge cases) that the code does not.

> **Antipattern to avoid:** answering from memory or from a quick grep of the code. If a knowledge layer exists, it is the source of truth for *intent*. Code is the source of truth for *current implementation*. They answer different questions.

### Write triggers — run `/keep-compile`

After or during non-trivial code changes in a repo that has `/knowledge/`:

- New feature or extension of an existing one
- Dependency / framework / runtime swap
- Topology change (new service, new boundary, new flow)
- Bug fix after an incident (runbook candidate)
- Configuration change that affects runtime behavior

### Drift triggers — run `/keep-check-drift`

Before merging a PR, before declaring a task complete, when the diff modifies code that any spec/ADR/runbook references via `related:` patterns.

### Idea triggers — run `/keep-idea`

The user proposes something but explicitly does NOT want to implement it now. Phrases: *"not now but"*, *"park this"*, *"jot this down"*, *"I'm thinking we should"*, *"don't lose this idea"*, *"come back to this later"*, *"thought experiment"*, *"what if we"*.

### When NOT to trigger

- Trivial edits (typo, formatting, single-line fix with no semantic content).
- Questions about general programming knowledge unrelated to this repo.
- Questions whose answer is obvious from a top-level README and don't touch architecture.

When in doubt, **lean toward triggering**. Under-triggering is the dominant failure mode.

---

## Core principles

1. **Memory, not narration.** Capture rationale, edge cases, rejected alternatives, operational learnings. Skip implementation walkthroughs that re-narrate the code.

2. **Knowledge has a consumer and an enforcement mechanism, or it is a lie.** A spec that no one reads and that no drift check verifies will silently diverge from reality. Every file in `/knowledge` has YAML frontmatter so `/keep-ask` can find it, `INDEX.md` can list it, and `/keep-check-drift` can match it against code. Files without frontmatter are unverified artifacts — fix them or delete them.

3. **Bidirectional flow.** Knowledge influences code (the agent consults it before answering or implementing); code influences knowledge (changes are classified and proposed back). Ideas are the third channel: half-formed thoughts captured durably, later promoted into specs/ADRs.

4. **Brownfield-first.** Work incrementally. Never require a full upfront documentation pass. Grow from real changes — or ingest pre-existing docs via `scripts/init.sh` + `/keep-compile ./folder/` with per-file approval.

5. **Conservative compilation.** Never invent intent. Stick to what the diff and existing docs actually support. The agent's compulsion to elaborate is the enemy of trustworthy knowledge.

6. **Minimal diffs.** When updating an existing knowledge file, change as little as possible. Preserve human wording verbatim.

7. **Ask before writing low-confidence content.** A diff shows *what*, rarely *why*. When `/keep-compile` is about to write a high-stakes field (rejected alternatives in an ADR, edge cases in a new spec, root cause in a runbook-tagged spec) and the diff doesn't unambiguously establish it, stop and ask in **batch mode** (multiple correlated questions in one turn). See `references/elicitation.md`.

8. **INDEX.md is derived, never authored.** It is regenerated from YAML frontmatter by `scripts/build_index.py`. If you find yourself editing INDEX.md by hand, you are introducing the drift this skill exists to prevent.

---

## Repository structure

```
/knowledge
├── docs/
│   ├── specs/             ← what the system should do (per-domain subdirs)
│   │                        - spec: behavioral spec
│   │                        - spec + tag:runbook: failure modes / operational
│   │                        - spec + tag:architecture: topology / boundaries
│   └── decisions/         ← ADRs (flat, ADR-NNNN-slug.md)
├── ideas/                 ← half-formed proposals (status: draft → promoted or deprecated)
└── INDEX.md               ← auto-generated by scripts/build_index.py
```

**Type vocabulary is deliberately small** — `spec`, `adr`, `idea`. Runbooks and  docs are *specs with a tag*, not separate types. v1 had four types and a `tasks/` folder for execution state; both were dropped. Execution state (active work in progress) belongs in your ticket tracker or in agent task lists, not in the durable knowledge layer — KEEP indexes things with a consumer and an enforcement mechanism, and ephemeral state has neither.

For per-file format with full YAML frontmatter schema, read `references/file_formats.md`.

For monorepo conventions (single `/knowledge/` at root, per-package subdirs under `specs/`), read `references/monorepo.md`.

---

## The seven commands

KEEP exposes a narrow surface: seven slash commands. One root, one bootstrap, five action verbs.

### `/keep` — root command / dashboard

Run with no args. Two modes, picked automatically by `scripts/status.py`:

- **If `/knowledge/` doesn't exist** — KEEP is uninitialized. The command tells the user to run `/keep-init` and stops. `/keep` is read-only by contract — it never bootstraps silently.
- **If `/knowledge/` exists** — prints a dashboard: counts of specs/ADRs/ideas, **anchor coverage %** (from `scripts/coverage.py`), and adaptive hints (e.g. *"anchor coverage in domain X is 0% — consider adding anchors so /keep-check-drift is deterministic"*). Then a one-line reminder of the action commands.

`/keep` is the only command an agent should reach for when uncertain *which* sub-command applies. It never writes anything; it routes.

### `/keep-init` — bootstrap (one-time per repo)

Use when the repo has no `/knowledge/`, or when `/keep` reports "uninitialized" and the user wants to proceed. Always asks for explicit consent before writing. Does, in order:

1. Runs `scripts/init.sh` — scaffolds `/knowledge/docs/{specs,decisions}/`, `/knowledge/ideas/`, INDEX.md. Detects monorepo. Appends KEEP snippet to AGENTS.md (or CLAUDE.md / .cursorrules).
2. Installs `SPEC-000-keep.md` from `references/templates/` into `/knowledge/docs/specs/keep/` — a self-spec describing KEEP's conventions inside the knowledge layer, so they survive the skill being uninstalled.
3. Runs `scripts/build_index.py` to refresh INDEX.md.
4. Mentions (but does NOT install) the optional CI / pre-commit setup. See `references/setup.md` step 5 for copy-paste templates.

Refuses to overwrite an existing `/knowledge/`. Re-running on an already-initialized repo is a no-op + status print.

### `/keep-ask <question>` — the only read command

Answer using `/knowledge/` with `[id]` citations. Reads `INDEX.md`, filters by frontmatter `description`/`tags`/`domain`, opens 1-5 relevant files, synthesizes. Read-only.

Two output shapes from the same command:

- **Synthesis** (default): answer with citations. Use for *"how does X work?"*, *"why did we choose Y?"*.
- **List-only** (when the user says *"just list paths"* or you're about to open the files yourself before implementing): same selection logic, returns file ids + paths + one-line descriptions, no synthesis.

**If the knowledge layer doesn't cover the question, say so explicitly.** Do not fall back to generic knowledge presented as repo truth — that is the antipattern this command exists to prevent.

### `/keep-compile [source] [--dry] [--migrate]` — the only write command

Two phases in one command:

1. **Observe** — classify the diff/source (`git diff`, branch, PR, tag, commit range, or a file/folder of pre-existing docs). Categorize each change as Feature / Architecture / Decision / Operational / Refactor. **Detect renames** in the diff and cross-reference existing anchors so a symbol rename surfaces as a proposed anchor update *before* it causes a drift failure at merge time. Output: suggested updates referenced by `id`. No writes yet.
2. **Compile** — apply each suggested update: create/modify files with valid YAML frontmatter, **propose anchor candidates from concrete values in the diff** (the default, not optional), run elicitation in batch for high-stakes fields, follow ADR supersession protocol. Then regenerate `INDEX.md` via `scripts/build_index.py` (which also refreshes the Backlinks section).

Pass `--dry` to stop after phase 1 ("show me what would change before I commit to it").

**Brownfield default is cordon-off, not ingest.** Passing a folder source writes a single cordon ADR declaring the folder out of scope for `/keep-ask` and `/keep-check-drift`. To migrate a specific legacy file into the knowledge layer, the user explicitly passes `--migrate` on that file — single-file only, no batch migration. The friction is the feature; see `references/brownfield.md` for the rationale (an unverified spec poisons every downstream query, so the default is "don't touch").

The observe+compile collapse into one command, and the new anchor / rename / cordon defaults, are the v4 simplifications. Earlier versions had separate `/keep-observe` and treated brownfield as ingest-by-default; both produced friction with no upside.

### `/keep-check-drift [source]` — enforcement (deterministic)

Verify that every `anchors:` entry in a spec's frontmatter still matches its target code. Backed by `scripts/check_drift.py` — a stdlib Python script with regex matchers per language (Go, Python, TypeScript) and per anchor kind (`const`, `function`, `test`, `manual`). **No LLM in the loop.** Exit code 1 blocks merge.

Anchored specs vs unanchored specs:
- A spec without an `anchors:` block contributes nothing to drift detection. Anchoring is opt-in per spec.
- A spec WITH anchors is enforced strictly: a renamed symbol, a changed const value, a skipped test all trigger failure.

Run from CLI or as a git pre-commit hook:

```bash
python3 <skill>/scripts/check_drift.py --changed   # only check files in git diff
```

See `references/file_formats.md` for the anchor schema (kinds, required fields, language matrix).

### `/keep-govern` — periodic hygiene

Detect cumulative entropy across the whole `/knowledge` base: stale files, duplication, contradicting ADRs without supersedes, oversize files, `TODO(KEEP)` markers, ideas older than 30 days in `draft`, specs without `related:` patterns, stray v1-style directories. Suggestions only. **Runs occasionally** — weekly at most.

> **Difference between `/keep-check-drift` and `/keep-govern`:** drift is correctness *now* on a specific diff (blocks merge). Govern is hygiene *over time* on the whole knowledge base (non-blocking). A file can pass drift but fail govern.

### `/keep-idea <free-form description>` — capture

Save a half-formed proposal to `/knowledge/ideas/<YYYY-MM-DD>-<slug>.md` with `type: idea`, `status: draft`. Search for prior art first — surface anything related. **Preserve the user's wording.** Promotion to a spec/ADR happens later through `/keep-compile`, with explicit approval. Ideas are NOT tasks (different lifecycle, different durability).

---

## YAML frontmatter is mandatory

Every file in `/knowledge/docs/` and `/knowledge/ideas/` starts with:

```yaml
---
id: SPEC-auth-jwt
title: "JWT validation"
description: "Behavioral spec for JWT issuer/audience validation, expiry handling, and refresh flow on /auth/* endpoints"
status: accepted               # draft | accepted | deprecated | superseded
type: spec                     # spec | adr | idea
domain: auth
tags: [auth, jwt, security]    # 'runbook' and 'architecture' are reserved tags
related:
  - adr:ADR-0014
  - test:internal/auth/*_test.go matching TestJWT*
  - code:internal/auth/jwt.go
---
```

The `description` field is the most load-bearing. Written as a *search snippet*, not a chapter heading. `/keep-ask` decides whether to load the file based on this string. Bad: `"Auth stuff"`. Good: `"Behavioral expectations for JWT validation: issuer/audience checks, expiry, secret rotation. Covers refresh flow and grace window."`.

The `related` field uses **convention-based references** — `test:internal/auth/*_test.go matching TestJWT*` is a pattern, not a hard-coded list of file paths. This is what makes drift detection work without breaking on rename.

Full schema, templates per type, supersession protocol, linking convention: `references/file_formats.md`.

---

## How the skill asks questions

KEEP commands ask the user for input in several situations: confirming `/keep-init`, choosing migrate vs cordon for a legacy doc, deciding what to do with contradicting claims during a migration, filling rejected-alternatives for a new ADR. The form of the question matters more than people think — open-ended questions get slow, unfocused answers; structured questions with explicit options get fast, decisive ones.

Apply this hierarchy:

1. **Binary decisions** (proceed / abort, e.g. `/keep-init` consent): yes/no with the default marked explicitly. `Scaffold /knowledge/ and append AGENTS.md? [y/N]` — the capital N is the default.

2. **Choice between 2-4 named options** (migrate vs cordon, classify as spec vs ADR vs idea): present them as a lettered or numbered list with a one-line description each and a marked default. Accept the letter, the keyword, or natural language — the user shouldn't need to remember a flag.

3. **Granular decisions in batch** (per-row verification of legacy claims, per-field ADR elicitation): one turn, multiple rows, each row offers the same small set of actions. Accept shorthand like `1b 2a 3c` or `all (a)`. Cap at ~5 rows per batch — beyond that, split.

4. **Free-form input** (ADR `## Context`, idea body, rationale prose): only when the user's exact words are the value. Always show a tentative draft alongside the question — *"Here's what I'd write based on the diff alone — does this miss anything?"* gets faster, better answers than *"What's the context?"*.

When in doubt, lean toward more structure. A migrate/cordon decision asked as *"What do you want to do with this file?"* will produce a slower, vaguer answer than the same question asked as *"(a) migrate, (b) cordon, default (b)"*. The cost of the structure is one extra sentence of skill prose; the benefit is the user can answer in one keystroke.

## Eliciting tacit knowledge

A diff shows *what changed*. It rarely shows *why*, what alternatives were rejected, what edge cases the author had in mind. That information lives in the user's head. KEEP captures it by *asking*, not by inferring.

This protocol applies primarily to `/keep-compile`. Three rules:

1. **Ask only when all three are true:** the field is high-stakes (ADR rejected alternatives, spec edge cases, runbook root cause), you have low confidence (the diff supports multiple explanations), and the field is being *created or substantially extended*.

2. **Batch mode for new ADRs and new specs.** Correlated fields. Ask all 2-3 questions in one turn (`ask_user_question`-style), then write in one pass. Reactive mode for runbooks and incremental updates.

3. **Cap at three questions per turn.** Show your tentative draft alongside the question — "Here's what I'd write based on the diff alone — does this miss anything?" gets faster better answers than open-ended interrogation.

If the diff or its commit messages already contain explicit evidence ("switched from KServe because of CRD complexity"), do NOT re-ask. Quote the commit verbatim in the file.

If the user declines to answer, omit the section and insert `<!-- TODO(KEEP): rejected alternatives not captured -->`. `/keep-govern` surfaces these later when context may be fresher.

---

## The standard workflow

```
First time on a repo:
  /keep                             ← dashboard. If /knowledge missing, points the user to /keep-init.
  /keep-init                        ← scaffold + install SPEC-000-keep + append AGENTS.md snippet (asks consent)
  /keep-compile ./docs/             ← cordon-off pre-existing docs (default — writes a cordon ADR, doesn't ingest)
  /keep-compile docs/auth/jwt.md --migrate   ← opt a specific legacy file into the knowledge layer

Every session, before answering:
  /keep-ask <user's question>       ← MANDATORY on read triggers (see above)

Every code change worth keeping:
  /keep-compile                     ← classify + propose anchors + write + regen INDEX in one shot
  /keep-compile --dry               ← (or: dry-run if you want to preview first)

Before merge:
  /keep-check-drift                 ← deterministic enforcement gate (exit 1 blocks merge)
                                      Best wired into CI / pre-commit — see references/setup.md step 5

Capture without commitment:
  /keep-idea <user's thought>       ← inbox

Periodic:
  /keep                             ← dashboard / status check (includes anchor coverage)
  /keep-govern                      ← hygiene, weekly at most
```

---

## AGENTS.md snippet — install in the repo

`scripts/init.sh` appends this to whichever entry file exists at the repo root (`CLAUDE.md`, `AGENTS.md`, `.cursorrules`). The triggering language is intentionally hard:

```md
## KEEP — Knowledge layer for this repository

`/knowledge/` is the authoritative source of truth for:
- WHAT this system does (specs)
- WHY it is built this way (ADRs)
- HOW it operates under failure (specs with tag `runbook`)
- WHERE components live and connect (specs with tag `architecture`)
- WHAT we've been thinking about but not yet committed to (ideas)

### Mandatory consultation before responding

Before answering ANY of these, run `/keep-ask <topic>` first:
- Questions about system behavior, design, or history
- Questions naming a feature, service, ADR, endpoint, or domain
- Requests to implement, modify, refactor, or remove existing behavior
- Any uncertainty about whether a decision or convention exists

If `/keep-ask` returns "no indexed knowledge", say so explicitly in your
answer — do NOT fall back to generic knowledge as if it were repo truth.

### After non-trivial code changes

Run `/keep-compile` (which classifies the diff and writes the updates in one shot) before declaring the work done.
A change that touches behavior, architecture, or operations without
updating `/knowledge` is incomplete.

### Before merging

Run `/keep-check-drift` on the diff. Drift exit-code 1 blocks merge.

### Periodic

`/keep-govern` weekly for hygiene.

### Capture, don't drop

Half-formed ideas → `/keep-idea "..."`. Don't lose them in chat.
```

---

## Anti-goals

KEEP must not become:

- a documentation generator that produces prose on demand
- a project management or ticketing system (use your tracker; KEEP only stores durable knowledge)
- a heavyweight SDD framework with upfront design ceremonies
- an AI wiki that drowns the repo in auto-generated text
- a workflow engine with lifecycle states and approval gates

If a request would push KEEP in any of these directions, push back. The value comes from staying small.

---

## Reference files

- `references/file_formats.md` — full YAML frontmatter schema, anchors block, templates per type, linking convention, ADR supersession protocol. Read when creating or updating any knowledge file.
- `references/monorepo.md` — monorepo layout and per-package routing for `specs/`. Read in a monorepo or when adopting on one.
- `references/brownfield.md` — cordon-off-by-default policy, `--migrate` workflow for opt-in per-file migration, classification heuristics. Read before invoking `/keep-compile` on a folder.
- `references/setup.md` — single source of truth for the AGENTS.md snippet, common adoption mistakes, the verification loop, and copy-paste templates for wiring `/keep-check-drift` into pre-commit / GitHub Actions. Read once at adoption.
- `references/templates/SPEC-000-keep.md` — the self-spec installed by `/keep-init` into `/knowledge/docs/specs/keep/`. Documents KEEP's conventions inside the knowledge layer so they outlast the skill.

---

## Anchor coverage — the silent KPI

Specs without anchors still work, but they sit outside the drift gate — they are claims without enforcement. The silent KPI of a KEEP-enabled repo is **what percentage of load-bearing code symbols (top-level constants, exported functions, named tests, routes) are referenced by at least one anchor**.

`scripts/coverage.py` reports this number per domain and per file. The `/keep` dashboard surfaces the aggregate. When `/keep-compile` writes a new spec, propose anchors *by default* — anchor proposal is part of writing the spec, not an optional add-on. The rule is "no anchor without evidence" (only propose anchors for values actually present in the diff), not "no anchor at all". See `commands/keep-compile.md` for the heuristic and `scripts/coverage.py` for the measurement.

## CI / pre-commit integration (recommended, not auto-installed)

`/keep-check-drift` is deterministic and CI-friendly — exit code 1 blocks merge. It becomes a real enforcement gate only when wired into your team's pre-commit hook or CI workflow. KEEP does NOT install hooks or workflows on the user's behalf; that kind of side effect is invasive. Instead, `references/setup.md` (step 5) carries copy-paste templates for both pre-commit and GitHub Actions. When `/keep-init` finishes, the agent mentions this once and lets the user opt in.

## Core insight

Code stores execution logic.

KEEP stores everything around it — architecture, rationale, operational learnings, evolving ideas — so the repository remains *understandable* as it grows. The value isn't "AI that writes code". It's "repositories that become living knowledge graphs an agent can keep in sync indefinitely".

Every commit can either deepen that knowledge or rot it. The skill exists to make deepening the default path.
