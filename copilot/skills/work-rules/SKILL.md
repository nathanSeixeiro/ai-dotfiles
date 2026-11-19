---
name: work-rules
description: Work rules for MyOrg devops: K8s/EKS/AKS, GitHub, Terraform, PR labels, commit trailer, clusters. Use before any ops, git, PR or infra change.
---

# Work Rules

## Kubernetes Clusters

### EKS (AWS)
- Use EKS MCP tools for READ-ONLY operations
- Also use `kubectl --context <cluster-name>` for READ operations when MCP lacks info
- NO write/mutate via MCP — no permission
- For changes: use kubectl with appropriate context
- Contexts: eks-dev, eks-hml, eks-prod

### AKS (Azure)
- Use kubectl for read operations
- For changes: `kubectl --context target-cluster`

## GitHub
- Use GitHub MCP for READ-ONLY operations (get files, list PRs, issues)
- For write operations (push, create PR, create branch): use `gh` CLI

## Repositories
- GitOps manifests (K8s kinds): `work/_devops/_argo/k8s-gitops`
- Cluster infrastructure (Terraform): `work/_devops/_terraform/k8s-infra`

## Terraform
- Local runs: `terraform init -backend=false` then `terraform validate` ONLY
- NEVER run `terraform apply` locally
- Plan and apply happen via CI/CD only

## Sub-agents
- Use sub-agents for multi-step processes and parallel work

## Applications
- Most apps are .NET
- Not a programmer — help with debugging, logs, and troubleshooting
- Focus on operational support, not code architecture

## Output — `~/work/__kiro-cli/`

| Folder | Content |
|--------|---------|
| `specs/` | Task specs (requirements + tasks + status) |
| `reports/` | Reports, analyses, audits |
| `temp/` | Temp files (terraform scratch, dumps, etc) |
| root | Checklists, inventories, loose docs |

## Communication
- Follow caveman-mode.md rules


## PR Labels (required)

Every PR created must have labels for identification:

### Application repos (e.g.: app-c, app-b, etc)
- `devops` — CI/CD, pipeline, Dockerfile, infra change
- `ci` — pipeline-specific changes

### Infrastructure repos (k8s-infra, k8s-gitops)
- Affected cluster: `eks-dev`, `eks-hml`, `eks-prod`, `aks-dev`, `aks-hml`, `aks-prd`
- Type: `security`, `migration`, `ci`
- Specific cloud resource (if applicable): `terraform`, `frontdoor`, `keyvaults`

### Charts/helm repos
- `migration` — cluster-to-cluster migration
- `eks-prod`, `aks-prd` — target cluster



## PRs — NEVER assume state

- NEVER treat an old PR as open/current. The user approves/merges in parallel.
- Always check the real state (`gh pr view`) before commenting on a PR.
- Don't say "PR #X is open/pending" without checking — it may already be merged.
- When creating a new branch, always branch from the up-to-date main (not an old stale branch).


## Commits — Co-author

- Author stays the user (the repo's git config). NEVER change `user.name` / `user.email` and never pass `--author`.
- Every commit the agent makes ends with this trailer, as its own paragraph (with git: a second `-m`):
  `Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>`
- Example: `git commit -m "feat(x): short description" -m "Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"`
- Applies to all repos and all agents. When asked to write a squash-merge message, keep the trailer in it.
- Do not add the trailer to commits the user wrote or asked to make as their own.
