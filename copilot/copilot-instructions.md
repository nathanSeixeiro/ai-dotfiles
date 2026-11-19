# Work Rules — DevOps MyOrg/MyOrg

## Kubernetes

### EKS (AWS)
- EKS MCP tools: read-only (get, list, describe).
- For reads, you may also use `kubectl --context <cluster>` when the MCP lacks the info.
- NEVER use MCP to write/mutate resources — no permission.
- Changes: use `kubectl` with the right context.
- Contexts: `eks-dev`, `eks-hml`, `eks-clt-prod`.

### AKS (Azure)
- `kubectl` for reads.
- Changes: `kubectl --context <target-cluster>`.

## GitHub
- GitHub MCP: read-only (files, PRs, issues).
- Write operations (push, create PR, create branch): use `gh` CLI.

## Repositories
- GitOps manifests (K8s): `work/_devops/_argo/k8s-gitops`
- Cluster infra (Terraform): `work/_devops/_terraform/k8s-infra`
- Reusable workflows (CI/CD): `my-org/devops-workflows`

## Terraform
- Locally only: `terraform init -backend=false` then `terraform validate`.
- NEVER run `terraform apply` locally.
- Plan and apply happen via CI/CD only.

## Sub-agents
- Use sub-agents for multi-step processes and parallel work.

## Applications
- Most apps are .NET.
- Focus on operational support (debug, logs, troubleshooting), not code architecture.

## PR Labels (required on every PR)

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

## File output
When generating specs, reports, audits, or temp files for this user, save them under `~/work/__kiro-cli/`:

| Folder | Content |
|--------|---------|
| `specs/` | Task specs (requirements + tasks + status) |
| `reports/` | Reports, analyses, audits |
| `temp/` | Temp files (terraform scratch, dumps, etc) |
| root | Checklists, inventories, loose docs |

## DNS (workflow `dns-update.yml`)
- Repo: `my-org/k8s-infra`
- Trigger: `gh workflow run dns-update.yml --repo my-org/k8s-infra -f action=<add|remove> -f environment=<Desenvolvimento|Homologacao|Staging|Producao> -f cloud="<Azure (AKS)|AWS (EKS)>" -f domains="<comma-separated fqdns>" -f record_type=<CNAME|A> -f dns_scope=<Interno|Externo|Ambos>`
- dev/hml: `dns_scope=Interno`. stg/prd: `dns_scope=Ambos`.
- **Production: ALWAYS ask the user before creating DNS.**
- Create DNS automatically when creating an HTTPRoute, except in production.

## AWS Account IDs / ECR
| Environment | Account ID | ECR |
|-------------|-----------|-----|
| dev | 111111111111 | 111111111111.dkr.ecr.us-east-1.amazonaws.com |
| hml | 222222222222 | 222222222222.dkr.ecr.us-east-1.amazonaws.com |
| prd | use secret `PRD_AWS_ECR` | — |

## Azure ACR
| Environment | Registry |
|-------------|----------|
| dev | acrdev.azurecr.io |
| hml | acrhml.azurecr.io |
| prd | acrprd.azurecr.io |

## Clusters
| Environment | EKS | Region | AKS | Region |
|-------------|-----|--------|-----|--------|
| dev | eks-dev | us-east-1 | aks-dvp | brazilsouth |
| hml | eks-hml | us-east-1 | aks-hml | brazilsouth |
| prd | eks-clt-prod | us-east-1 | aks-pro | brazilsouth |

## Deploy — EKS vs AKS
- `develop`/`homolog`: deploy via **GitOps** (update-image-tag -> ArgoCD sync) on both clouds.
- EKS `staging`/`main` (prd): **direct** deploy via kubectl/helm.
- AKS: **all** environments (dev/hml/stg/prd) use GitOps.
- Sonar runs on `develop` only. appsettings transform runs on `staging`/`main` only.

For the full onboarding guide for new apps (checklists, Dockerfile, CI/CD pipelines, Key Vault), use the `app-onboarding` skill.
For complex multi-step tasks, use the `spec-workflow` skill.

## Communication style
Always use the `caveman-mode` skill in ALL responses, by default, without the user needing to ask every prompt.
