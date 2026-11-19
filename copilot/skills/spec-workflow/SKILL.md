---
name: spec-workflow
description: Structured requirements -> tasks -> approval -> execution flow for complex DevOps/infra tasks (3+ steps or touching multiple resources). Use when the user asks to plan, or the task spans multiple steps/resources.
---

# Spec Workflow

## When to activate
- Task has 3+ steps or touches multiple resources.
- Simple tasks (1-2 steps) = execute directly, no spec.

## Phases
1. **Requirements** — List what needs to happen. Bullet points. Constraints. Acceptance criteria.
2. **Tasks** — Break into discrete, ordered steps with dependencies.
3. **⏸ Approve** — STOP. Show the tasks. Wait for the user to confirm before executing.
4. **Execute** — Run the tasks. Use sub-agents for parallel work when possible.

## Rules
- Save the spec in `~/work/__kiro-cli/specs/<short-name>.md`.
- File format: `feature-name.md` (kebab-case, no date).
- The file contains: requirements + tasks + status.
- Mark tasks as completed as they finish.
- No design phase. No fluff. Requirements -> Tasks -> Do.

## Spec file format

```markdown
# <Spec Name>
Date: YYYY-MM-DD

## Requirements
- [ ] requirement 1
- [ ] requirement 2

## Tasks
- [ ] task 1
- [ ] task 2
- [ ] task 3

## Status
- Started: YYYY-MM-DD
- Completed: pending
```

## Behavior
- After showing the tasks, wait for explicit confirmation ("ok", "go ahead", "approved" or similar).
- If the user says "skip the spec" or the task is clearly simple, execute directly.
- Sub-agents handle parallel execution of independent tasks.
