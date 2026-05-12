# /observe — worked examples

Use these to calibrate classification when a diff is ambiguous. The principle running through all of them: stick to what the diff actually shows, and resist the temptation to narrate.

## Example 1 — clean feature addition

**Diff (summary):** new file `auth/jwt_refresh.py` implements a token refresh flow. `auth/jwt.py` gets a new `refresh()` method. Tests added.

**Good `/observe` output:**

```
Detected changes:

Feature:
- Added JWT refresh flow

Suggested knowledge updates:
- knowledge/docs/specs/auth/jwt.md  (add: refresh flow requirements and acceptance criteria)
```

**Bad output (over-classified):**

```
Detected changes:

Feature:
- Added JWT refresh flow
Architecture:
- Authentication subsystem extended
Operational:
- New failure modes possible

Suggested knowledge updates:
- knowledge/docs/specs/auth/jwt.md
- knowledge/docs/architecture/auth.md
- knowledge/docs/runbooks/jwt-refresh-failures.md
- knowledge/docs/decisions/ADR-NNNN-refresh.md
```

The diff is a single behavioral addition. There is no topology change, no rejected alternative, no observed failure mode. A spec update is sufficient.

## Example 2 — architectural shift with a real decision

**Diff (summary):** removed direct FastAPI → Triton calls. Introduced Ray Serve as an intermediary. New `serve/` directory. The PR description explains they considered KServe and BentoML.

**Good `/observe` output:**

```
Detected changes:

Architecture:
- Introduced Ray Serve as inference orchestration layer between API and Triton

Decision:
- Ray Serve chosen over KServe and BentoML (rationale in PR description)

Suggested knowledge updates:
- knowledge/docs/architecture/inference.md  (update: new flow API → Ray Serve → Triton)
- knowledge/docs/decisions/ADR-NNNN-ray-serve.md  (create: choice + rejected alternatives)
```

This is exactly when an ADR earns its place: a non-trivial choice with stated alternatives.

## Example 3 — pure refactor, no semantic change

**Diff (summary):** renamed `utils.py` → `helpers.py`, split a 400-line file into three smaller modules. No behavior change. All tests pass unchanged.

**Good `/observe` output:**

```
Detected changes:

No semantic changes detected. Refactor only.

Suggested knowledge updates:
- None
```

Refactors that preserve behavior do not require knowledge updates. Resist the urge to record the rearrangement.

## Example 4 — operational learning from an incident fix

**Diff (summary):** added retry logic with exponential backoff for JWKS fetches. Commit message references an incident where Auth0 rate-limited the service.

**Good `/observe` output:**

```
Detected changes:

Feature:
- Added retry with exponential backoff for JWKS fetches

Operational:
- JWKS rate-limiting from Auth0 observed in production (incident context in commit)

Suggested knowledge updates:
- knowledge/docs/specs/auth/jwt.md  (add: retry behavior in requirements)
- knowledge/docs/runbooks/jwks-rate-limit.md  (create: symptoms, cause, mitigation)
```

Real production incidents are exactly what runbooks are for. Capture them once, when the fix is fresh.

## Example 5 — ambiguous diff

**Diff (summary):** a 50-file diff touching frontend, API, and inference layers. No clear narrative in the commit messages.

**Good `/observe` output:**

```
Detected changes:

The diff spans multiple layers and the intent is not clear from the commits alone.
Before classifying, I need to understand what this change is trying to accomplish.

Could you summarize the goal of this change, or point me at the PR description / issue?
```

When in doubt, ask. Speculative classification poisons the knowledge layer.

## Example 6 — observing a PR (not just a diff)

**Source:** `/observe PR#142` — a feature branch with 12 commits ahead of `main`. The PR title is "Add JWT refresh + JWKS caching" and the description mentions Auth0 rate-limiting.

**Good `/observe` output:**

```
Detected changes (from PR#142, 12 commits):

Feature:
- Added JWT refresh flow (commits a3f21b, c8d442)
- Added JWKS in-memory cache with TTL (commits e1f9ab, ...)

Architecture:
- Introduced JWKS cache layer between auth middleware and Auth0 (commit e1f9ab)

Operational:
- Retry-with-backoff on JWKS fetch, motivated by rate-limit incident
  (commit message: "fix: retry JWKS after we got rate-limited last Tuesday")

Suggested knowledge updates:
- knowledge/docs/specs/auth/jwt.md  (add: refresh requirements, JWKS cache behavior)
- knowledge/docs/architecture/auth.md  (update: new cache layer in flow)
- knowledge/docs/runbooks/jwks-rate-limit.md  (create — commit message confirms a real incident)

Notable commit messages:
- "fix: retry JWKS after we got rate-limited last Tuesday"  ← operational evidence
- "decided against pre-warming the cache because the rotation window is short"  ← ADR signal
```

The PR mode gives `/observe` two things a bare diff doesn't: the **narrative arc** across commits, and **rationale fragments** in commit messages. Surface those quotes verbatim — they are the cheapest source of decision context the user has to verify.

The commit about "decided against pre-warming" is a strong signal for an ADR. `/compile` should pick it up.

## Example 7 — observing a folder of existing docs (brownfield ingestion)

**Source:** `/observe ./old-docs/` — a folder of pre-existing markdown files (architecture diagrams, design notes, an `auth-design.md`, a `deployment-runbook.md`).

**Good `/observe` output:**

```
Detected ingestion candidates (from ./old-docs/, 8 files):

Likely specs:
- ./old-docs/auth-design.md → maps to knowledge/docs/specs/auth/jwt.md
  (covers JWT validation, issuer/audience checks, expiry handling)

Likely architecture:
- ./old-docs/system-overview.md → maps to knowledge/docs/architecture/overview.md
- ./old-docs/inference-flow.md → maps to knowledge/docs/architecture/inference.md

Likely runbooks:
- ./old-docs/deployment-runbook.md → maps to knowledge/docs/runbooks/deployment.md
  (already structured as symptoms/causes/mitigation — direct fit)

Likely ADRs (require elicitation):
- ./old-docs/why-ray-serve.md  ← mentions alternatives but no explicit rejection rationale
- ./old-docs/database-choice.md  ← mentions Postgres vs MongoDB but no consequences captured

Unclear (need user input):
- ./old-docs/notes-from-meeting-march.md  ← mixed content, possibly multiple targets

No action suggested for:
- ./old-docs/old-todo-list.md  ← appears to be ephemeral task state, not durable knowledge
```

Ingestion mode is **classification only** — `/observe` never migrates content. It surfaces the mapping so `/compile` can run the elicitation protocol per file and migrate with user approval. Unclear or ambiguous files get flagged for user decision rather than guessed at.

## General rules

- One source, one set of suggestions. Do not chain "and this might also mean…" inferences.
- If a category is empty, omit it. Do not write `Operational: None.`
- **Quote** commit messages or doc fragments that contain rationale signals rather than paraphrasing them — they are evidence the user can verify.
- When the source is large and heterogeneous, prefer asking the user to narrow the scope over producing a sprawling classification.
- For ingestion of existing doc folders: never auto-migrate. Always classify and let `/compile` handle the migration with elicitation.
