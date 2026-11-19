---
name: jira-cards
description: PROJ Jira board card rules. Use whenever a task finishes — create/move/update a Task card on the PROJ board (example.atlassian.net) with a short what-done description and links, title pattern [PROJECT] topic. Also use when the user points to an existing task to adopt/reformat it.
---

# Jira — Card Creation and Movement

Project: **PROJ** (`example.atlassian.net`). Access via Atlassian MCP.

## Auto-card on task finish (ALWAYS)

Every agent, when it FINISHES a task, records it on the PROJ board. No exception unless user says "no card".

On finish:
1. **Check first** — related card already exists, OR user pointed to an existing task.
   - If yes → **reuse that card**. Move/update it. Reformat title+description to the patterns below.
   - If no → **create** a new card in BACKLOG.
2. **Move** — if work started/finished, move to `In Progress`. Never auto-`Done`.
3. **Update** — set title + description per patterns below (short what-done + links).

Before any move/update: confirm real card state in Jira via MCP (don't assume).

## Scope of action (STRICT)

Only touch these three statuses:

- **BACKLOG** — where I create new cards.
- **In Progress** — when work starts/finishes.
- **Done** — ONLY when the user explicitly says so.

DO NOT touch: `SCHEDULED`, `FROZEN`, `OVERDUE`, `THIRD-PARTY PENDING`.

## Allowed transitions

```
(new) -> BACKLOG -> In Progress -> Done
```

- New card enters **BACKLOG**.
- Move to **In Progress** when execution starts or on finish.
- Move to **Done** ONLY on explicit user notice. Never automatic, even if PR merged.

## Types

- Only `Task`.

## Title

- Pattern: `[PROJECT] topic`.
  - `PROJECT` = app/repo/area (uppercase).
  - `topic` = concise imperative description.
  - E.g.: `[AKS-PRD] PodDisruptionBudget for apps with >=2 replicas`
  - E.g.: `[APP-D] remove AppDynamics agent (pre-OTel migration)`
  - E.g.: `[GITOPS] onboard new app namespace overlay`
- If an existing/related card has a different title → **rename it to this pattern**.

## Description

- Short: what was done (1-2 lines).
- Links: issue, PR, and any solution/problem ref.
- Format:
  ```
  <1-2 lines: what was done>

  PR: <main PR url>
  Issue: <issue url, if any>
  Ref: <solution/problem/extra link, optional>
  ```
- Multiple PRs → list main one, mention others in a single line.
- Reusing an existing card → reformat its description to this layout.

## Rules

- Never mark Done on your own — wait for "you can close / mark it done".
- Assignee: keep as is (don't reassign unless asked).
- Before moving/closing a card, confirm real state in Jira (don't assume) — same as PRs.
- Link card <-> delivery: reference PROJ-XXX when it makes sense.
- User points to a task OR asks to work on one → adopt that card, reformat to patterns, don't create a duplicate.

## Pending validation

Confirm exact status/transition names against the board via MCP on first op (e.g.: "In Progress" vs "Em andamento", "Done" vs "Feita"). Adjust this file if it differs.
