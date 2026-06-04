---
description: Detect drift between code and /knowledge/ — deterministic anchor verification, CI-friendly (exit 1 on drift).
argument-hint: [--spec SPEC-id] [--changed] [--verbose]
---

Input: `$ARGUMENTS`

This is a thin entry point for `/keep-check-drift`. The full, canonical contract lives inside the skill so it travels with both the plugin and skill-only installs:

**`skills/KEEP/references/commands/keep-check-drift.md`**

Read that file and follow it exactly against the current repository. It is the single source of truth for this command — make any behavior change there, never here.
