---
description: One-shot knowledge update — classify a diff or pre-existing source, write/update files, regenerate INDEX. Use `--dry` to stop after classification.
argument-hint: [source] [--dry]
---

Use the `keep` skill in **compile mode**. `/keep-compile` is the single command for "I made changes, update knowledge." It dispatches on the *kind* of source it's given:

| Source kind | Examples | Behavior |
|---|---|---|
| **Code diff** | no args (working diff), `feature/auth-refresh`, `PR#142`, `main..HEAD`, `HEAD~3..HEAD` | Normal ingest flow: classify the diff, write/update spec+ADR, propose anchors, regen INDEX. Unchanged from earlier KEEP versions. |
| **Pre-existing doc folder** | `./docs/`, `./old-docs/`, `./wiki/` | Asks the user: cordon the whole folder (write one cordon ADR, default), or walk through file-by-file with verification? |
| **Pre-existing doc file** | `docs/auth/jwt.md`, `ARCHITECTURE.md`, `RUNBOOK.md` | Asks the user: migrate this file into `/knowledge/` (with verification against current code), or cordon it (add to or create the cordon ADR)? |

Pass `--dry` to stop after phase 1 (useful for "show me what would change before I commit to it"). `--dry` applies to all three source kinds.

Source resolution order when no argument is given:

1. **Working diff** — `git diff` against the branch's merge base.
2. **Last commit** — `git diff HEAD~1` as final fallback.

Whenever the source is *ambiguous* (e.g., the user said *"compile this"* with no path), confirm what they mean before doing anything. *"Compile the working diff, or did you mean a specific file/folder?"* is a one-second cost that saves a wrong write.

When a PR or branch is given, pull commit list with messages — rationale often hides in commit prose ("switched from KServe because of CRD complexity"). Quote those messages verbatim; do not paraphrase.

---

## Phase 1 — Observe

1. **Read `INDEX.md`** to see what's already in the knowledge layer and which `id`s exist.
2. **Detect renames** before classifying. Scan the diff for symbols that disappear from one file and reappear under a new name in the same file (e.g. `const TOKEN_TTL = …` deleted, `const JWT_TOKEN_TTL = …` added with the same value). For each rename detected, check existing specs' `anchors:` blocks — if any anchor references the old symbol, surface the rename as a **proposed anchor update** in phase 1's output. The user confirms in phase 2. Catching renames before they hit `/keep-check-drift` saves a CI failure.
3. **Categorize each change** into one of:
   - **Feature** — new/modified behavior → spec change
   - **Architecture** — topology/boundary change → architecture-tagged spec
   - **Decision** — choice with rejected alternatives → new ADR
   - **Operational** — failure mode learned → runbook-tagged spec
   - **Refactor** — no semantic change → no knowledge update
4. **For each non-refactor change, list affected knowledge files by `id`** and what kind of update they need. Be specific about *what*, not just *which*.
5. **Surface rationale-bearing commit messages verbatim.** Phrases like "because", "instead of", "we tried", "this breaks", "incident", "rejected" are gold for ADR alternatives and runbook causes — quote them, do not paraphrase.
6. **Refactor-only diff?** Say so explicitly, propose ZERO updates, do not regenerate INDEX. A pure rename or find-and-replace with no behavior change is execution detail, not knowledge — capturing it is the antipattern KEEP exists to prevent.

Output of phase 1:

```
Detected changes:

Feature:
- <one-line behavioral change>

Decision:
- <one-line — candidate for new ADR>

Rename detected:
- TOKEN_TTL → JWT_TOKEN_TTL in internal/auth/jwt.go
  → SPEC-auth-jwt::token_ttl anchor needs update (symbol)

Suggested knowledge updates:
- [SPEC-auth-jwt] update edge cases section to reflect <change>
- [SPEC-auth-jwt] anchor `token_ttl`: symbol TOKEN_TTL → JWT_TOKEN_TTL
- [ADR-NNNN] create new ADR for <decision> (number resolved in phase 2)
```

Reference files by their `id` (from frontmatter), not by raw paths. The `id` is stable; paths can move.

If `--dry` was passed, stop here.

---

## Phase 2 — Compile (write the files)

For each suggested update from phase 1:

- **New file** — create with full YAML frontmatter (see `references/file_formats.md`). The `description` field must be a *search snippet*, not a chapter heading. The `related` field uses convention-based patterns (`code:internal/auth/*_test.go matching TestJWT*`), not hard-coded paths.
- **Updated file** — make the smallest possible diff. Preserve human-written rationale verbatim. Update the `related` field if cross-references changed.
- **Anchor update from a rename** — apply the symbol rename to the affected `anchors:` entry. Do not invent additional anchors during this step; only fix the one that drifted.
- **New ADR** — `ls decisions/` first, pick the next free `ADR-NNNN`. Run **batch elicitation** (2-3 correlated questions in one turn) for rejected alternatives and consequences if the diff doesn't establish them. Quote any commit messages that already capture rationale instead of re-asking.
- **ADR supersession** — when a new ADR replaces an old one: new file gets `supersedes: [ADR-NNNN]`; old file's `status` becomes `superseded` and gets a `## Superseded by` section appended. Body of the old ADR is never edited.
- **Scaffolding a brand-new domain** — when phase 1 introduced a domain with no prior `specs/<domain>/` content, propose two companion specs, not one:
  1. The behavioral spec for the feature that triggered the new domain (e.g. `specs/billing/invoices.md`).
  2. An architecture-tagged stub sketching how the domain fits (`specs/billing/topology.md` with `tags: [billing, architecture]`).

  Topology can be a stub (paragraph + `<!-- TODO(KEEP) -->` markers for unknowns) — the point is to plant the file with a search-snippet description so `/keep-ask` can route topology questions correctly. Skip this only when the new domain is a one-file utility with no topology to describe; note the omission in final output.

- **Propose anchor candidates — by default, not as an afterthought.** When writing a NEW spec, or substantially extending an existing one, scan the diff for **concrete values bound to identifiers** and propose them as anchors *as part of the spec, not as an optional add-on*:

  - integer/float/string literals assigned to a top-level `const`/`var` → `kind: const`
  - newly added or modified function signatures → `kind: function`
  - newly added `Test*` / `test_*` / `it('…')` declarations → `kind: test`
  - new HTTP routes / handlers if you can detect them syntactically → `kind: manual` with a `notes:` pointing to the route registration

  For each candidate, draft the entry directly into the spec's `anchors:` block, then surface the list to the user in one batch:

  > I drafted 4 anchor candidates for this spec. Reply `keep all`, `keep 1,3,4`, or `edit` to walk through them.

  **Hard rule: do not invent anchors for values that aren't in the diff.** If the diff doesn't establish a binding, no anchor — the spec body documents the claim, drift detection does not enforce it. Specs without anchors still work; they just sit outside the drift gate. The point of anchors is *enforceability*, not coverage theater.

### Brownfield — ask, don't assume

When the source is **pre-existing documentation** (a Markdown / RST / TXT / ADoc file or a folder of them, anywhere in the working tree that is NOT under `/knowledge/`), the right action is ambiguous and depends on whether the operator has verified the file's currency. So the agent **asks** instead of guessing.

#### Single legacy file

If the source is a single doc file (e.g. `docs/auth/jwt.md`, `ARCHITECTURE.md`):

1. Read the file briefly (titles + first paragraph), so the question can be specific.
2. Ask the user:

   > `docs/auth/jwt.md` looks like an existing doc. Two options:
   >
   > **(a) Migrate** — I'll extract the claims, verify each against current code, ask you about anything that disagrees or has no binding, and write a clean spec into `/knowledge/`. Verified literal values become anchors automatically.
   >
   > **(b) Cordon** — I'll mark this file as legacy / out of scope for `/keep-ask` and `/keep-check-drift`. The file stays where it is; nothing imported into `/knowledge/`.
   >
   > Which do you want?

   Default to **cordon** if the user is silent or unclear — it is the no-op-equivalent safe choice.

3. On **(a) migrate** → run the verify-and-write sequence below.
4. On **(b) cordon** → append the file path to an existing cordon ADR if one covers this folder, or write a new one (`ADR-NNNN-legacy-docs-cordoned.md`) referencing this single file.

#### Folder of legacy docs

If the source is a folder (e.g. `./docs/`, `./old-docs/`):

1. Walk the folder once to count files, just to make the question concrete.
2. Ask the user:

   > `./docs/` has 12 markdown files. Two options:
   >
   > **(a) Cordon the folder** — one ADR that declares the whole folder legacy / out of scope. Files stay where they are. Fast, safe, low-trust default.
   >
   > **(b) Walk through file-by-file with verification** — I'll go through each file, ask if you want to migrate it (with verification), cordon it, or skip it. Better for repos with a small number of genuinely current docs you want to keep authoritative.
   >
   > Which do you want?

   Default to **(a) cordon the folder** — safer when in doubt.

3. On **(a)** → write the cordon ADR per the template in `references/brownfield.md`. Update an existing cordon ADR for the same path instead of duplicating.
4. On **(b)** → loop: for each file, run the single-file ask above. Skip is a third option that records the file as "intentionally not in /knowledge" without a cordon ADR entry.

#### The migrate sequence (whichever way we got here)

A migrate is **not** a verbatim copy. The agent extracts durable claims, checks them against current code, and asks the user about anything uncertain — so the resulting spec carries only facts that are still true today.

1. **Classify** — propose target type per `references/brownfield.md` (spec / spec+runbook / spec+architecture / ADR / idea) from headings and body. Confirm with the user.

2. **Extract claims** — scan the file for *verifiable* claims, the kind that could become anchors:

   - literal values bound to identifiers ("TOKEN_TTL is 5 minutes", "batch size 4")
   - function or method signatures ("`ValidateToken(token, secret, issuer)` returns Claims")
   - endpoints / routes ("`GET /auth/refresh` returns 200")
   - test names ("verified by `TestJWTExpiry`")
   - file path references ("see `internal/auth/jwt.go`")

   Prose claims (rationale, alternatives, history) pass through as text. Don't try to verify "we chose Postgres because of ACID guarantees" against code — it has no binding.

3. **Verify** — for each verifiable claim, run a deterministic check against the current code (grep the symbol, compare value/signature). Mark each claim:

   - ✓ **matches** — file claim is consistent with current code
   - ✗ **contradicts** — file claim disagrees with current code
   - ? **unverifiable** — no code binding found

4. **Report and ask** — present the verification table; ask only about ?-rows and ✗-rows:

   ```
   docs/auth/jwt.md — verification:

   ✓ TOKEN_TTL = 5min                    matches const TOKEN_TTL in internal/auth/jwt.go
   ✓ ValidateToken(token, secret, iss)   matches func signature
   ✗ uses HS256 signing                  code uses ES256 — claim is outdated
   ? GET /auth/refresh returns 200       no obvious route binding — confirm or drop?
   ? "rate limit 100 req/min"            no code binding — confirm or drop?
   ```

   Batch the questions (max 3-4 per turn). For ✗-rows, propose: drop the claim, replace with current truth, or keep with `<!-- TODO(KEEP): outdated -->` marker.

5. **Write the spec** with the decisions applied:

   - ✓ claims become text *and* anchor candidates in frontmatter (verification gave us bindings for free)
   - ✗ claims dropped or rewritten per user choice
   - ? claims kept-or-dropped per user choice; kept ones may get `kind: manual` anchors if the user can point at a non-code source (Terraform module, Confluence page)
   - Provenance comment near the top: `<!-- Migrated from docs/auth/jwt.md on 2026-05-19; verified against commit <sha>; claims confirmed/dropped per session with @user -->`

6. **Anchors and INDEX** — fold the new spec into the regular finishing sequence: anchors drafted in step 5, INDEX.md regenerated.

Why ask instead of assuming: a folder of unverified docs at scale (10+ files) is not workable manually — the operator can't realistically verify them all without a long session. Cordon is the honest default when in doubt. Migration is the per-file escape hatch for the small number of legacy files that *are* worth verifying. Asking puts the decision where it belongs (with the user) without making them remember a flag. See `references/brownfield.md` for the full reasoning.

### Regenerate `INDEX.md` (mandatory last step)

```bash
python <skill-path>/scripts/build_index.py knowledge/ --strict
```

The script walks `/knowledge/docs/` and `/knowledge/ideas/`, parses YAML frontmatter, emits a deterministic table-based `INDEX.md` including the auto-generated **Backlinks** section (the reverse map of `related:` references). **Never hand-edit `INDEX.md`** — if the layout is wrong, fix the script. `--strict` exits with code 1 if any file has invalid/missing frontmatter; surface those to the user as a `/keep-govern` backlog item.

---

## Hard rules

- Minimal diffs. If you're rewriting a paragraph, stop and reconsider.
- Frontmatter is mandatory on every file you create. Files without frontmatter are unverified artifacts and they will lie.
- Verify filesystem state before sequential or set-based claims (next ADR number, whether a domain exists, whether a file is present). Check, don't assume.
- Never invent rejected alternatives, edge cases, or root causes. If the diff and elicitation can't establish them, omit the section and insert `<!-- TODO(KEEP): ... -->` for `/keep-govern` to surface later.
- Never auto-promote idea content into specs/ADRs. Surface and ask.
- Folders without `--migrate` produce a cordon ADR, not an ingestion. `--migrate` is single-file only.
- After writing the knowledge files, **always run `build_index.py`**. INDEX.md must be derived from frontmatter, including the Backlinks section.

## Summary output

```
Phase 1 (observe):
- Detected 3 changes: 2 features, 1 decision, 1 rename
- Suggested updates: 1 new ADR, 1 new spec, 1 spec update, 1 anchor update

Phase 2 (compile):
Created:
- [ADR-0015] knowledge/docs/decisions/ADR-0015-dual-secret-rotation.md
- [SPEC-auth-refresh] knowledge/docs/specs/auth/refresh.md (4 anchors proposed, all accepted)

Updated:
- [SPEC-auth-jwt] edge cases (grace window now applies); anchor token_ttl symbol renamed TOKEN_TTL → JWT_TOKEN_TTL

Regenerated:
- INDEX.md (12 entries: 7 specs, 4 ADRs, 1 idea; Backlinks section refreshed)
```
