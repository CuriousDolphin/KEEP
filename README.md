# KEEP — Knowledge Engine for Engineering Persistence

[![skills.sh](https://skills.sh/b/CuriousDolphin/KEEP)](https://skills.sh/CuriousDolphin/KEEP)

KEEP is a memory layer for **agentic, spec-driven software development**.

It doesn’t change how code is written.  
It changes how agents understand what they are working on.

---

## Why KEEP exists

Modern agentic development has a structural problem:

- code evolves continuously
- specs and decisions are implicit or scattered
- agents repeatedly re-derive context from scratch
- architectural intent gets lost between iterations

The result is not just poor documentation — it’s **context amnesia**.

Agents can see the code, but they don’t reliably understand:

- why it exists
- what decisions shaped it
- what constraints are active
- what was already tried and rejected

KEEP exists to fix this.

---

## The core idea

KEEP introduces a persistent memory layer inside the repository:

- **Code** → execution layer
- **/knowledge** → structured system memory
- **Agent** → reasoning layer

Instead of forcing agents to infer context from code alone, KEEP gives them a **shared, evolving source of truth about intent**.

---

## What KEEP is actually for

KEEP is designed specifically for:

> agentic workflows where multiple reasoning steps happen over time on the same codebase

In practice, it helps agents (and humans!):

- maintain architectural continuity across sessions
- understand historical decisions without re-reading entire repos
- avoid re-solving already solved design problems
- keep spec, code, and intent aligned over time

---

## How it works

KEEP maintains a living knowledge base inside the repo:

- architectural decisions (ADRs)
- system specifications
- operational runbooks
- design constraints and tradeoffs
- historical evolution of the system

Everything is structured so an agent can reconstruct **intent**, not just implementation.

---


## Installation

The skill lives under `skills/KEEP/` (Agent Skills format: `SKILL.md` + optional `references/`).

### skills CLI

```bash
npx skills add CuriousDolphin/KEEP --skill keep -a claude-code -y
```

Install globally:

```bash
npx skills add CuriousDolphin/KEEP --skill keep -g
```

Verify:

```bash
npx skills add CuriousDolphin/KEEP --list
```

---

### Claude Code plugin marketplace

KEEP is also distributed as a plugin:

```text
/plugin marketplace add CuriousDolphin/KEEP
/plugin install keep@keep-knowledge
```

Validate locally:

```bash
claude plugin validate .
```

---

## When to use KEEP

KEEP is used when agents need to preserve or recover context about the system they are working on.

Typical workflow:

1. **/observe** → analyze changes in code and intent signals
2. **/compile** → update the knowledge layer minimally and consistently
3. **/retrieve** → bring only relevant context into agent working memory
4. **/govern** → detect drift, duplication, or stale knowledge

---

## /knowledge structure

/knowledge
├── docs/
│   ├── specs/
│   ├── decisions/
│   ├── architecture/
│   └── runbooks/
├── tasks/
└── INDEX.md

This is not documentation.

It is a **persistent memory layer optimized for agent reasoning**.

---

## The four commands

| Command | Purpose | Writes files? |
|--------|--------|--------------|
| `/retrieve` | Fetch relevant knowledge for context | No |
| `/observe` | Analyze changes (PR, Commits, folder with docs..) + intent | No |
| `/compile` | Update knowledge base incrementally | Yes |
| `/govern` | Maintain consistency and reduce entropy | No |

---

## Philosophy

KEEP is not about better documentation.

It is about enabling agents to maintain **continuous understanding of a system over time**.

---

## In short

KEEP is useful when:

- multiple agent sessions work on the same codebase
- context is too large to re-load every time
- architectural intent matters as much as implementation
- spec-driven development is the primary workflow

It is not about writing more documentation.

It is about making sure agents never lose the meaning behind the system.
