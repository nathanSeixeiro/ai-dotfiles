# Environment map

## Repos (local)

| What | Path |
|---|---|
| GitOps | `~/work/devops/k8s-gitops` (a second clone exists at `~/work/_gitops`: check `git -C <path> log -1` and `git remote -v`, use the freshest, always `git pull`) |
| Infra (Terraform) | `~/work/devops/terraform/k8s-infra` |
| Reusable workflows | `~/work/project/devops-workflows` |
| App repos | `~/work/project/<repo>`; remote `my-org/<repo>` |

`work-rules.md` still lists `work/_devops/...` paths: they do not exist on this machine.

## GitOps layout

- `{Aks|Eks}/kubernetes/overlays/<env>/ns-<app>-<env>/<component>/` : `deployment.yaml`, `service.yaml`, `configmap.yaml`, `external-secret.yaml`, `kustomization.yaml` (image tag lives here), `pdb.yaml`
- `{Aks|Eks}/kubernetes/overlays/<env>/ns-<app>-<env>/scheduled-scaling/` : KEDA ScaledObject
- `{Aks|Eks}/kubernetes/base/external-secrets/<env>/` : ClusterSecretStores (`azure-kv-<app>`)
- `{Aks|Eks}/kubernetes/argocd/<env>/{projects,applications}/` : AppProject and Application (QA under `argocd/hml/.../qa/`)
- `docs/PROJECTS.md` : catalog project -> cloud -> env -> namespace (regenerate: `scripts/list-projects.sh`)
- `.github/instructions/*.md` : conventions (envoy-gateway, keda, observability-stack, kubernetes-manifests)

Did a deploy happen? The pipeline bumps the tag with commit `chore(<app>): bump <env> to <tag>` by `argo-image-bumper[bot]`. Check: `git log --oneline -5 -- <overlay path>`. Then ArgoCD sync, then the image on the live pod.

## Infra layout

- KV: `Aks/terraform/keyvaults/environments/<env>/kv-<app>.tf` (QA files live inside `environments/hml/`). Vault name `kv-<d|h|q|p>-<app>`, max 24 chars. Terraform provisions the vault, RBAC (ESO reader/admin) and placeholder secrets.
- Clusters, node pools (incl. Windows pools), network: `Aks/` and `Eks/` trees.
- Terraform plan/apply happen in CI only.

## Environments

See `app-onboarding.md` tables (envs, clusters, ACR/ECR, KV prefixes). QA shares the hml cluster, isolated by namespace + KV.

kubectl contexts: EKS `eks-dev`, `eks-hml`, `eks-prod`; AKS `aks-dvp`, `aks-hml`, `aks-prd`; local `kind-clt`. `eks-prod` is the only correct EKS prd context name.

## Facts that look like failures but are not

- dev/hml KEDA cron: min 0 / max 1, on 7am off 9pm. Pods absent at night = by design. Check the ScaledObject before calling it an outage.
- Windows pools exist only in hml and prd (apps named `*-win` or Windows images). Windows pods start slowly and log via LogMonitor to stdout.
- Many deployments use `dnsPolicy: None` + corporate DNS; others use `ClusterFirst` with `dnsConfig` commented out. Check each deployment, do not assume.

## Observability

- App logs: `kubectl logs` (use `--previous` after a restart); Windows IIS pods: stdout via LogMonitor (EventLog + IIS + log4net).
- Splunk O11y (OTel auto-instrumentation; OTLP to node hostIP:4318), CloudWatch MCP for EKS.
- ArgoCD: sync/health/last operation error per Application.
