# MyOrg — Claude Code project memory

Personal config for Claude Code, ported from Kiro CLI (`../kiro/`). Full detail for every topic below lives in `claude/skills/*` (loaded on demand by description match) and `claude/agents/*` (subagent definitions). This file is the always-loaded summary.

## Communication style

Generalist core, always on (no skill trigger needed, applies to every agent and the main session): short, direct, code over prose, no filler, no pleasantries. End every task with 1-2 lines: what was skipped, what was not checked, risk. Infra/ops-specific scope-discipline (blast radius, smallest-complete-change) is a separate on-demand skill: `caveman-mode` — it only loads for actual ops/infra work, not general chat.

## Kubernetes

- EKS (AWS): read via EKS MCP tools or `kubectl --context <cluster>`; write only via `kubectl`. Contexts: `eks-dev`, `eks-hml`, `eks-prod`.
- AKS (Azure): `kubectl --context <cluster>` for reads and changes. Contexts: `aks-dvp`, `aks-hml`, `aks-prd`.
- NEVER mutate prd (`aks-prd`, `eks-prod`) directly — read-only, hand the exact command to the user.
- No `kubectl delete` via any agent, anywhere.

## GitHub

- Read via GitHub MCP (files, PRs, issues). Write (push, PR, branch) via `gh` CLI.
- Never push to main/master. Branch from up-to-date main. Check `gh pr view` for real state before saying anything about a PR — never assume it's still open.
- Every PR needs labels (app repos: `devops`/`ci`; infra repos: cluster + type; charts: migration/target).
- Every agent commit ends with its own trailer paragraph: `Co-authored-by: Claude <noreply@anthropic.com>`. Never change `user.name`/`user.email`, never `--author`.

## Terraform

- Locally: `terraform init -backend=false` then `terraform validate` only. NEVER `apply`/`destroy`/`plan` locally — CI/CD only. Plan review comes from `gh pr checks` / `gh run view --log`.

## Repositories

- GitOps manifests (K8s): `my-org/k8s-gitops`
- Cluster infra (Terraform): `my-org/k8s-infra`
- Reusable CI/CD workflows: `my-org/devops-workflows`

## Output files

Write specs, reports, checklists and temp files under `~/work/__kiro-cli/`: `specs/` (task specs), `reports/` (reports/analyses), `temp/` (scratch, cleaned up after the task), root (checklists/inventories).

## CI/CD workflow paths

Multi-app repos: one workflow per app, `on.push.paths` scoped to that app's own folder/Dockerfile/workflow file only. Never a catch-all glob. Full rule: skill `workflow-path-triggers`.

## Jira (PROJ board)

Every agent records finished work on the PROJ board (reuse existing card or create in BACKLOG, move to In Progress, Done only on explicit user word). Full rules: skill `jira-cards`. Closing a task: skill `task-handoff`.

## Multi-step work

3+ steps, 2+ subagents, or 2+ repos: track with the `TodoWrite` tool before starting (item = outcome + owner + proof; gates are items too). Full discipline: skill `todo-discipline`. For 3+ step work needing a written spec first: skill `spec-workflow`.

## Subagents

Use the `Task` tool to spawn the named subagents in `claude/agents/` for orchestrated multi-step work (see `devops-lead`, `dev-lead` for routing). Independent roles run in parallel. A lead never does the specialist work itself.

## Delegation

Main session stays general-purpose — no default agent. For a single small/direct ask, just do it. Otherwise spawn via `Task`: ops/infra request -> `devops-lead`; dev/code request -> `dev-lead`; vague ask or 3+ tasks -> `project-manager` first for a spec, then route per its plan. Those leads spawn the rest of the specialists themselves — don't spawn a specialist directly from the main session, go through its lead.

## Applications

Most apps are .NET (legacy + modern); also JS/TS (Backstage portal), Go (Crossplane functions), bash/PowerShell. Detect the stack from the repo — don't assume.

## Secrets

Never print, echo, log, or commit secret values. Never read `.env`, PEM/key files, credential caches unless the user explicitly says so. List names/keys only (`az keyvault secret list`, not `show`). Mask anything that looks like a secret in command output.

## Production safety

Always confirm with the user before: Terraform apply, any prod change, DNS change, merge, or any destructive op. These are human-only.

## Not ported from Kiro

- `aws-specialist` (AWS read-only consultant subagent) — dropped entirely, including every other agent's "spawn aws-specialist for AWS questions" section.
- `gmud-create` skill (corporate Jira ITSM change-ticket creation) — dropped, not reusable outside that specific process.
