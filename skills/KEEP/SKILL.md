---
name: keep
description: "Living /knowledge layer for a code repo — specs, ADRs, ideas, with runbook and architecture as tags. CONSULT whenever the user asks how the system works, why a decision was made, what conventions exist, or whether a change is safe — even without naming KEEP. ALSO trigger on non-trivial changes that affect behavior/architecture/operations in a /knowledge-enabled repo, doc ingestion, parked ideas ('not now but', 'what if we'), and PR drift checks. SKIP for generic programming Q&A, library docs, pure mechanical refactors, and repos without /knowledge."
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

## The six commands + one bootstrap script

KEEP exposes a narrow surface: six slash commands (one root, five action verbs) plus a one-time bootstrap script.

### `/keep` — root command / dashboard

Run with no args. Two modes, picked automatically by `scripts/status.py`:

- **If `/knowledge/` doesn't exist** — KEEP is uninitialized. The command asks the user *"Want me to run `scripts/init.sh` to scaffold `/knowledge/` and append the AGENTS.md snippet?"*. On yes → runs the bootstrap script, then re-shows the dashboard. On no → two-line explanation of KEEP and stops.
- **If `/knowledge/` exists** — prints a dashboard: counts of specs/ADRs/ideas, anchor coverage %, and adaptive hints (e.g. *"anchor coverage is low — consider adding anchors so /keep-check-drift is deterministic"*). Then a one-line reminder of the five action commands.

`/keep` is the only command an agent should reach for when uncertain *which* sub-command applies. It never writes anything; it routes.

### `scripts/init.sh` — one-time bootstrap (NOT a slash command)

Invoked by `/keep` on first run, or run directly from the repo root: `bash <skill-path>/scripts/init.sh`. Scaffolds `/knowledge/`, detects monorepo layout, scans for pre-existing docs (READMEs, ARCHITECTURE, RUNBOOKS, ADR folders), and appends the KEEP workflow snippet to whichever instruction file exists (`AGENTS.md` / `CLAUDE.md` / `.cursorrules`). Refuses to overwrite an existing `/knowledge/`. Bootstrap is a one-time job — making it a slash command misled users into thinking it was part of the daily loop.

### `/keep-ask <question>` — the only read command

Answer using `/knowledge/` with `[id]` citations. Reads `INDEX.md`, filters by frontmatter `description`/`tags`/`domain`, opens 1-5 relevant files, synthesizes. Read-only.

Two output shapes from the same command:

- **Synthesis** (default): answer with citations. Use for *"how does X work?"*, *"why did we choose Y?"*.
- **List-only** (when the user says *"just list paths"* or you're about to open the files yourself before implementing): same selection logic, returns file ids + paths + one-line descriptions, no synthesis.

**If the knowledge layer doesn't cover the question, say so explicitly.** Do not fall back to generic knowledge presented as repo truth — that is the antipattern this command exists to prevent.

### `/keep-compile [source] [--dry]` — the only write command

Two phases in one command:

1. **Observe** — classify the diff/source (`git diff`, branch, PR, tag, commit range, or a folder of pre-existing docs). Categorize each change as Feature / Architecture / Decision / Operational / Refactor. Output: suggested updates referenced by `id`. No writes yet.
2. **Compile** — apply each suggested update: create/modify files with valid YAML frontmatter, run elicitation in batch for high-stakes fields, follow ADR supersession protocol, handle ingestion with provenance comments. Then regenerate `INDEX.md` via `scripts/build_index.py`.

Pass `--dry` to stop after phase 1 ("show me what would change before I commit to it"). Default invocation runs both phases.

The two phases are one command because in practice nobody runs observe without then running compile — splitting them produced friction with no upside. Earlier versions of KEEP had separate `/keep-observe` and `/keep-compile`; v2.2 collapsed them.

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
  /keep                             ← dashboard. If /knowledge missing, offers to run init.sh.
  /keep-compile ./docs/             ← migrate ingestion candidates with approval

Every session, before answering:
  /keep-ask <user's question>       ← MANDATORY on read triggers (see above)

Every code change worth keeping:
  /keep-compile                     ← classify + write + regen INDEX in one shot
  /keep-compile --dry               ← (or: dry-run if you want to preview first)

Before merge:
  /keep-check-drift                 ← deterministic enforcement gate (exit 1 blocks merge)

Capture without commitment:
  /keep-idea <user's thought>       ← inbox

Periodic:
  /keep                             ← dashboard / status check
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

- `references/file_formats.md` — full YAML frontmatter schema, templates per type, linking convention, ADR supersession protocol. Read when creating or updating any knowledge file.
- `references/monorepo.md` — monorepo layout and per-package routing for `specs/`. Read in a monorepo or when adopting on one.
- `references/brownfield.md` — heuristic catalog for doc scanning during `init.sh` and folder-ingestion via `/keep-compile ./folder/`. Read when migrating pre-existing docs.
- `references/setup.md` — single source of truth for the AGENTS.md snippet, common adoption mistakes, the verification loop. Read once at adoption.

---

## Core insight

Code stores execution logic.

KEEP stores everything around it — architecture, rationale, operational learnings, evolving ideas — so the repository remains *understandable* as it grows. The value isn't "AI that writes code". It's "repositories that become living knowledge graphs an agent can keep in sync indefinitely".

Every commit can either deepen that knowledge or rot it. The skill exists to make deepening the default path.
