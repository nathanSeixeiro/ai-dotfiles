---
name: spec-workflow
description: Project conventions for multi-step spec work — when to write a spec, where it lives, naming. Use when a request has 3+ steps or touches multiple resources, before starting work.
---

# Spec Conventions

For 3+ step work, write a spec file before implementing instead of just diving in. This file = project conventions only (no native `/spec` command in Claude Code — track phases with the TodoWrite tool and the spec file below).

## When

- 3+ steps or multiple resources → write a spec at `~/work/__kiro-cli/specs/<feature>.md` first
- 1-2 steps → execute direct, no spec

## Rules

- Spec = requirements → design → tasks → user approval → run, tracked as a todo list (`TodoWrite`) once approved.
- Output path: `~/work/__kiro-cli/specs/`
- Filename: `<feature>.md` (kebab-case, no date)
- Caveman-style output always
- Use the Task tool to spawn subagents for parallel independent tasks
