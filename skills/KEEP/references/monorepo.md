# Monorepo support — details and examples

The principle is summarized in `SKILL.md`: one `/knowledge/` at the root, with package-level subdirectories for `specs/` and `runbooks/`. This file covers the rationale, the full layout, command behaviors, and the decision tree for cross-package content.

## Why a single zone

Most knowledge in a monorepo is inherently cross-cutting:

- A shared auth library is used by three services. Its spec is one spec, not three.
- The deployment pipeline touches all packages. Its runbook is one runbook.
- A decision like "we use Postgres for everything" applies wherever data is stored.

Per-package knowledge zones force artificial duplication of all of the above, and require the agent to scan multiple zones for context — exactly what KEEP is trying to avoid. A single root-level zone with package-aware subdirectories keeps `/keep-retrieve` simple and lets cross-package relationships emerge naturally through the **Related** links.

## Full layout

```
/                                  ← repo root
├── knowledge/
│   ├── docs/
│   │   ├── specs/
│   │   │   ├── auth-service/             ← specs for the auth-service package
│   │   │   │   └── jwt.md
│   │   │   ├── inference-api/            ← specs for the inference-api package
│   │   │   │   └── pipeline.md
│   │   │   └── shared/                   ← specs spanning multiple packages
│   │   │       └── api-conventions.md
│   │   ├── decisions/                    ← flat, sequential
│   │   │   ├── ADR-0001-monorepo-tooling.md
│   │   │   ├── ADR-0002-auth0.md
│   │   │   └── ADR-0003-ray-serve.md
│   │   ├── architecture/                 ← flat, one file per area
│   │   │   ├── overview.md               ← cross-cutting topology
│   │   │   ├── auth-service.md
│   │   │   └── inference-api.md
│   │   └── runbooks/
│   │       ├── auth-service/
│   │       │   └── jwt-rotation.md
│   │       ├── inference-api/
│   │       │   └── gpu-oom.md
│   │       └── shared/
│   │           └── postgres-connection-pool.md
│   ├── tasks/
│   └── INDEX.md
├── packages/
│   ├── auth-service/
│   ├── inference-api/
│   └── shared/
└── ...
```

## Command behaviors in a monorepo

### `/keep-retrieve <topic>`

Searches the single zone. When the topic matches a package name, results from that package's subdirectories come first; cross-cutting results from `shared/`, flat ADRs, and `architecture/overview.md` come after, clearly labeled:

```
Relevant context for "auth":

Auth-service package:
- knowledge/docs/specs/auth-service/jwt.md
- knowledge/docs/architecture/auth-service.md
- knowledge/docs/runbooks/auth-service/jwt-rotation.md

Cross-cutting:
- knowledge/docs/decisions/ADR-0002-auth0.md
```

When the topic does not match a package name, search proceeds normally across all subdirectories and the user gets results grouped by category (specs / decisions / architecture / runbooks) without per-package grouping.

### `/keep-observe`

When classifying a diff, infer which package(s) are affected from the file paths in the diff:

- `packages/auth-service/src/...` → `auth-service`
- `packages/inference-api/src/...` → `inference-api`
- `packages/auth-service/src/...` AND `packages/inference-api/src/...` in the same diff → cross-package

Suggested update paths go under the package subdirectory for `specs/` and `runbooks/`. ADRs and architecture suggestions always reference the flat directories.

Example output for a cross-package diff:

```
Detected changes:

Feature:
- Updated shared API response format (affects auth-service and inference-api)

Architecture:
- Cross-cutting: changed the API conventions both services follow

Suggested knowledge updates:
- knowledge/docs/specs/shared/api-conventions.md  (update: new response envelope)
- knowledge/docs/architecture/overview.md  (update: response flow)
```

### `/keep-compile`

Writes spec and runbook updates under the affected package's subdirectory; writes ADRs and architecture updates flat. When a change spans multiple packages:

- **Prefer `shared/`** for specs and runbooks. A "shared API response format" spec belongs in `specs/shared/`, not duplicated in each package directory.
- **Mention the affected packages** in the file's **Related** section: *"Applies to: auth-service, inference-api"* (this is a custom Related label, but works fine as long as it's consistent).
- **ADRs that affect multiple packages** are flat. Their title and content make scope clear.

### `/keep-govern`

Runs across the whole zone but groups results by package subdirectory in the output:

```
Stale knowledge detected:

auth-service:
- knowledge/docs/specs/auth-service/api-keys.md  (last touched 11 months ago)

inference-api:
- (no stale files)

shared / flat:
- knowledge/docs/decisions/ADR-0003-ray-serve.md  (worth re-reading; relevant code changed twice this quarter)
```

## Decision tree — where does this content go?

| Content describes... | Goes in... |
|---|---|
| A single package's feature behavior | `specs/<package>/<feature>.md` |
| Behavior shared by multiple packages (e.g. API conventions, error format) | `specs/shared/<topic>.md` |
| A decision that affects only one package | `decisions/ADR-NNNN-<slug>.md` (flat — scope clear from content) |
| A decision that affects multiple packages | `decisions/ADR-NNNN-<slug>.md` (flat) |
| Internal topology of a single package | `architecture/<package>.md` |
| Cross-package topology (how packages relate) | `architecture/overview.md` |
| A failure mode in one package | `runbooks/<package>/<failure-mode>.md` |
| A failure mode that crosses packages (e.g. shared DB exhaustion) | `runbooks/shared/<failure-mode>.md` |

When the right location is genuinely unclear, `/keep-compile` asks rather than guessing:

> *"This spec describes how the API responds to malformed JSON, which applies to both auth-service and inference-api. Put it in `specs/shared/` (treat as cross-cutting) or in `specs/auth-service/` since that's where the change originated?"*

## `INDEX.md` structure in a monorepo

The root `INDEX.md` adds a `## Packages` section near the top:

```md
# INDEX

## Packages
- auth-service — specs/auth-service/, runbooks/auth-service/, architecture/auth-service.md
- inference-api — specs/inference-api/, runbooks/inference-api/, architecture/inference-api.md
- shared — specs/shared/, runbooks/shared/, architecture/overview.md

## Domains

### Auth
- knowledge/docs/specs/auth-service/jwt.md
- knowledge/docs/decisions/ADR-0002-auth0.md
- knowledge/docs/runbooks/auth-service/jwt-rotation.md

### Inference
- knowledge/docs/architecture/inference-api.md
- knowledge/docs/specs/inference-api/pipeline.md
- knowledge/docs/decisions/ADR-0003-ray-serve.md

## Entities
- JWT
- Auth0
- Ray Serve
- Triton

## Flows
- authentication flow
- inference pipeline
```

This makes "show me everything about a package" and "show me everything about a domain" both one-glance operations.

## Single-package fallback

In a non-monorepo, the structure collapses naturally:

- `specs/<package>/` becomes `specs/<area>/` — same convention, "area" instead of "package".
- No `## Packages` section in `INDEX.md`.
- No `shared/` subdirectory needed; all specs and runbooks are under their respective `<area>` directories.

All command behaviors apply unchanged. The monorepo conventions add complexity only where complexity exists.
