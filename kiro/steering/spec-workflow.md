---
inclusion: always
name: "Spec Conventions"
---

# Spec Conventions

Use native `/spec` for spec work. This file = project conventions only.

## When

- 3+ steps or multiple resources → `/spec new <name>`
- 1-2 steps → execute direct, no spec

## Rules

- Native `/spec` owns workflow (requirements → design → tasks → approve → run)
- Output path: `~/work/__kiro-cli/specs/`
- Filename: `<feature>.md` (kebab-case, no date)
- Caveman-style output always
- Sub-agents for parallel independent tasks
