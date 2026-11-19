---
inclusion: manual
name: "Todo Discipline"
---

# Todo Discipline

Mandatory for leads (`devops-lead`, `dev-lead`) and `project-manager`. Recommended for every specialist on 3+ step work. On top of `caveman-mode.md`.

Tool: `todo_list` (Kiro CLI). No `todo_list` tool in the runtime (Kiro Crew): keep the same list in `~/work/__kiro-cli/temp/todo-<task>.md`, rewrite it on every state change, and paste the current list in each progress message. Delete the file when the task is finished (temp cleanup rule).

## Why

Todo list = the plan you can be held to. It stops skipped gate steps, forgotten handoffs and "done" claims without evidence.

## Protocol

1. **Start**: request has 3+ steps, 2+ subagents or touches 2+ repos -> create the list BEFORE any action. 1-2 step work: no list.
2. **Resume**: if a list already exists for the task, load it first. Never start a second list for the same task.
3. **Item = outcome + owner + proof.** Format: `<verb> <thing> [owner: <agent|user|CI>] -> proof: <command/PR/verdict>`.
   - good: `terraform validate on keyvaults/dev [owner: terraform-specialist] -> proof: validate exit 0`
   - bad: `do terraform stuff`
4. **Gates are items.** Every gate step is its own item: qa, review, rules-validator, plan-in-CI, user sign-off. A gate not on the list is a gate that gets skipped.
5. **One item in progress at a time** per agent. Parallel subagents: mark each item in progress when spawned, complete when its report is verified.
6. **Complete only with evidence.** Mark done after you verified the proof (command output, `gh pr view`, validator verdict). A subagent claim is not proof.
7. **Blocked = stay open.** Add the blocker and owner in the item text (`blocked: waiting user merge PR #N`). Never delete or silently drop an item.
8. **New work found = new item.** Add it. Do not do hidden work outside the list. Scope creep goes to the list as "proposed" and waits for the user.
9. **Before the final answer**: re-read the list. Every item is done with proof, or open with a stated reason. Report open items in the close line.
10. **Prod / apply / DNS / merge / destructive** items: always a separate item owned by `user`. Never auto-complete.

## Template (copy the shape)

```
1. Map scope: repos, envs, blast radius [owner: lead] -> proof: scope note
2. <implementation item> [owner: <specialist>] -> proof: <build/validate output>
3. Gate: qa-agent tests [owner: qa-agent] -> proof: PASS line
4. Gate: semantic_reviewer [owner: semantic_reviewer] -> proof: APPROVE
5. Gate: rules-validator [owner: rules-validator] -> proof: PASS
6. Open PR with labels [owner: lead] -> proof: gh pr view
7. User review/merge [owner: user] -> proof: user says merged
8. Update PROJ card [owner: project-manager|lead] -> proof: card comment
```

## Anti-patterns

- List written after the work is done.
- Items like "investigate" with no proof.
- Everything marked done at the end in one batch.
- Subagent says PASS, lead ticks the box without reading the verdict.
