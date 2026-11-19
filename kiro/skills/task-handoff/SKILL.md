---
name: task-handoff
description: Write a closing/handoff note as a comment on the PROJ card when a task ends, so a new short session can resume from the card. Use when the user says encerrar, fechar chamado, nota de encerramento, handoff, passar para nova sessão, or when a task just finished (PR open, GMUD created, RCA delivered).
---

# Task Handoff (closing note on the PROJ card)

Goal: a new session resumes from the card, not from 100 turns of history. Card = control point (steering `jira-cards.md`).

## Step 1 - Find the card
- User gave a key (PROJ-123): use it.
- Else search: `searchJiraIssuesUsingJql` in project PROJ by the task topic. One clear match: use it. Several or none: ask the user. Never create a duplicate.
- Confirm real card state with `getJiraIssue` before writing (don't assume status).

## Step 2 - Collect facts (verify, don't remember)
- PRs: `gh pr view <n> --json state,mergedAt,url` for each PR of the task. State comes from the command, not from memory.
- Runs/apply/GMUD: check real state if the note mentions them.
- Pending items from the todo list: open ones with blocker + owner.

## Step 3 - Comment (`addOrEditJiraIssueComment`)
Short, caveman, max ~12 lines. Format:

```
Encerramento: <1 line: what was delivered>
Estado: <PR #N merged|open|CI failing | GMUD-XXX created | etc, as verified>
Pendente: <item> [owner: user|CI|agent] - <blocker>
Próximo passo: <1 line, concrete first action for the next session>
Contexto p/ retomar: <repos, branch, env, key paths/commands, 1-3 lines>
Links: PR <url> | Issue <url> | Ref <url>
```

Rules:
- No secret values, tokens or personal data.
- No Done. Never transition the card to Done on your own (only on explicit user word). Move to `In Progress` only per `jira-cards.md`.
- Update the card description `PR:` / `Issue:` lines if a new link appeared.
- If the comment already exists for this stop, edit it instead of adding another.

## Step 4 - Close
Reply with: card key + link, 1 line of what was written, and "abra sessão nova; retome lendo o card <KEY>".

## Resume (new session)
Read the card: `getJiraIssue` (description) + `listJiraIssueComments` (last `Encerramento:` comment). Start from `Próximo passo`. Re-check PR/run state with `gh` before acting.
