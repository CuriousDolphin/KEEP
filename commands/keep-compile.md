---
description: One-shot knowledge update — classify a diff or pre-existing source, write/update files, regenerate INDEX. Use `--dry` to stop after classification.
argument-hint: [source] [--dry]
---

Input: `$ARGUMENTS`

This is a thin entry point for `/keep-compile`. The full, canonical contract lives inside the skill so it travels with both the plugin and skill-only installs:

**`skills/KEEP/references/commands/keep-compile.md`**

Read that file and follow it exactly against the current repository. It is the single source of truth for this command — make any behavior change there, never here.
