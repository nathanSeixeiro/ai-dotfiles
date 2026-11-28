# Claude Code — personal config

## Communication style

Short, direct, code over prose, no filler, no pleasantries. End every task with 1-2 lines: what was skipped, what was not checked, risk.

## Kubernetes

No fixed clusters configured yet on this machine. When working with a cluster:
- Read via `kubectl --context <context>`.
- Never mutate anything that looks like production without explicit confirmation first.
- No `kubectl delete` without explicit confirmation.

## GitHub

- Read via `gh`/GitHub MCP if available. Write (push, PR, branch) via `gh` CLI.
- Never push directly to main/master. Branch from up-to-date main.
- Check `gh pr view` for real current state before saying anything about a PR — never assume it's still open/merged.
- Every commit/PR ends with the attribution trailer given in the system reminder for this session.

## Terraform

- Locally: `terraform init -backend=false` then `terraform validate` only. Never `apply`/`destroy`/`plan` against real infra locally — run those through CI/CD, or only with explicit confirmation if run by hand.

## Production safety

Always confirm with the user before: Terraform apply, any prod-like change, DNS change, merge, or any destructive op. These are human-only.

## Subagents

Multi-step dev or ops work can be orchestrated instead of done inline: `dev-lead` / `devops-lead` break work into tasks and spawn the specialists in `agents/` (dev-agent, qa-agent, rules-validator, terraform-specialist, k8s-troubleshooter, idp-dev, platform-engineer, project-manager). For a single small/direct ask, just do it yourself — no need to route through a lead.

## Secrets

Never print, echo, log, or commit secret values. Never read `.env`, PEM/key files, credential caches unless explicitly asked to. Mask anything that looks like a secret in command output.
