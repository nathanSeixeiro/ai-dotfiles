# Terraform: repos, state, pipelines

Facts read from the repos on 2026-11. Re-check with `git -C <repo> log -1` and the repo README before relying on them.

## Repos

| Repo | Local path | Scope |
|---|---|---|
| `k8s-infra` | `~/work/devops/terraform/k8s-infra` | AKS + EKS clusters, Key Vaults, Front Door, observability, Backstage module |
| `aws-infra` | `~/work/devops/terraform/aws-infra` | AWS resources per app: S3, IAM, ElastiCache, SG. One `.tf` per app per account |
| `terraform-gh-env` | `~/work/devops/terraform/terraform-gh-env` | GitHub Environments (+ secrets, protection rules) per app repo. `projects/<repo>/main.tf` |

`work-rules.md` still lists `work/_devops/_terraform/...` paths. They do not exist on this machine. Use the table above.

## k8s-infra layout

```
Aks/terraform/
  kubernetes/   modules/{observability,backstage}   environments/{dev,hml,prd}   state: <env>-kubernetes.tfstate
  frontdoor/    nonprod/ (dev+hml+qa)  prd/                                      state: nonprod-frontdoor.tfstate
  keyvaults/    environments/{dev,hml}  (QA files live inside hml/)               state: <env>-keyvaults.tfstate
Eks/terraform/
  environments/{dev,hml,prod}   modules/{cluster,karpenter,envoy-gateway,observability,cloudwatch-observability,velero}   tests/
Eks/iam/   Aks/backstage/{templates,catalog}   scripts/migration/   .kiro/{steering,skills,specs}
```

Apply order (Aks): `kubernetes/` first, then `frontdoor/` (consumes Envoy Gateway IP) and `keyvaults/` (consumes ESO identity).

## Pipelines (GitHub Actions, OIDC)

| Pipeline | Trigger path | Envs |
|---|---|---|
| `terraform-kubernetes.yml` | `Aks/terraform/kubernetes/**` | dev, hml (matrix) |
| `terraform-frontdoor-nonprod.yml` | `Aks/terraform/frontdoor/nonprod/**` | nonprod |
| `terraform-frontdoor-prd.yml` | `Aks/terraform/frontdoor/prd/**` | prd |
| `terraform-keyvaults.yml` | `Aks/terraform/keyvaults/**` | dev, hml (matrix) |
| `terraform-plan.yml` (PR) | `Aks/terraform/**` | auto-detects changed projects, plans only those |

Flow: PR -> plan comment. Merge to main -> Validate -> Plan -> **Approve (GitHub Environment)** -> Apply. Never apply locally. When a new project/path is added, the pipeline trigger paths must be updated too (`workflow-path-triggers.md`).

`aws-infra`: PR -> plan comment; merge -> apply behind environment `review`.

## AWS accounts (region us-east-1)

| Env | Account | State bucket (aws-infra) |
|---|---|---|
| dev | 111111111111 | `corp-aws-tf-state` (`dev/aws-infra.tfstate`) |
| hml | 222222222222 | `corp-aws-tf-state-hml` |
| prd | 333333333333 | `corp-aws-tf-state-prd` |
| network | 444444444444 | Transit Gateway, CloudFront/ALB |

## Known findings (verify before acting)

- `k8s-infra` tracks a plan file: `Aks/terraform/kubernetes/environments/prd/tfplan-tempo-observab`. A plan file can embed secret values. Treat as a finding: propose removal + `.gitignore` for `tfplan*` via PR; do not open or print it.
- `Aks/terraform/kubernetes/environments` has dev/hml in the README but a `prd` dir exists: check which pipeline covers it before touching.

## Where to find more

- `k8s-infra/.kiro/steering/` (terraform-azure, terraform-aws, aws-eks-infrastructure, aws-networking, aws-security, project-conventions)
- `k8s-infra/.kiro/skills/` (`terraform-planning`, `import-infrastructure`, `platform-sre-review`)
- gitops repo `.github/instructions/` (`terraform.instructions.md`, `terraform-azure.instructions.md`, `azure-verified-modules-terraform.instructions.md`)
