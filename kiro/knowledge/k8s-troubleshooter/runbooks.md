# Runbooks

Always: context printed, namespace set, read-only first.

## Triage table

| Symptom | Check | Common causes |
|---|---|---|
| Pod `Pending` | `describe pod` events; ScaledObject | KEDA scaled to 0 (dev/hml night); no node capacity; Windows pool full; nodeSelector/taint; quota; PVC |
| `CrashLoopBackOff` | `logs --previous`; exit code in `describe` | 137 = OOMKilled (raise memory request/limit); missing env/secret key; wrong port; app exception at startup; bad connection string |
| `ImagePullBackOff` / `ErrImagePull` | image + tag in kustomization; registry | tag does not exist; wrong registry for env (ACR dev/hml/prd, ECR accounts per env); pull credentials; build failed so tag never pushed |
| Running, not Ready | probes in deployment | probe path/port wrong (8080 only for .NET 8+, else 80); slow start (Windows); dependency down |
| Works then 5xx / timeouts | logs, `top`, events | CPU throttling (requests too low), pool exhaustion, DB slow, OOM restarts |
| 404 / 502 / 503 at edge | HTTPRoute, Gateway, Service, Endpoints | selector/port mismatch, empty endpoints, wrong hostname, DNS CNAME target, cert |
| Config change not applied | ArgoCD sync; `kustomize build`; pod age | not merged/synced; ConfigMap/Secret env does not hot-reload: pod needs restart (dev/hml ask, prd explicit OK) |
| ExternalSecret `SecretSyncedError` | `describe externalsecret`; ClusterSecretStore status | CSS not ready; KV missing (infra PR); RBAC missing; KV firewall; secret name `--` vs env `__`; key missing in KV |
| ArgoCD OutOfSync / Degraded | Application status, last operation message | drift; invalid manifest; `resource ... is not permitted in project` = AppProject `namespaceResourceWhitelist` missing the kind (e.g. PodDisruptionBudget) |
| Deploy "did nothing" | `gh run view --log-failed`; gitops `git log` for the bump commit | pipeline failed; `update-image-tag` skipped because kustomization path not found (it only warns); ArgoCD not synced |

## Internal DB / hostname / communication runbook

Seeded from team experience. Refine after real cases.

Typical errors: `Name or service not known`, `No such host is known`, `Temporary failure in name resolution`, SQL `A network-related or instance-specific error`, Oracle `ORA-12154`, `ORA-12170`, `ORA-12543`, generic connect timeouts.

1. WHAT host does the app use? Look in the ConfigMap (plain vars) and the app repo config binding. For secrets (connection strings in KV) do NOT read the value: ask the user, or infer from the error message in logs. Is it a short name (no domain) or an FQDN?
2. Internal DB servers usually need the `.corp.local` suffix. Short name without `.corp.local` = most common cause of DNS failure.
3. Pod DNS config: in the deployment read `dnsPolicy` and `dnsConfig` (nameservers, searches, ndots). `dnsPolicy: None` without the right `searches` = short names never resolve. `ClusterFirst` = depends on cluster DNS forwarding for corporate zones.
4. Prove it from inside the pod (dev/hml read-only OK, prd ask): `cat /etc/resolv.conf`, `nslookup <host>` and `nslookup <host>.corp.local`. Linux TCP test without tools: `bash -c '</dev/tcp/<host>/<port>'`. Windows pods: `Resolve-DnsName`, `Test-NetConnection`. Image without a shell/tools: `kubectl debug` needs a yes in chat (dev/hml); in prd it is denied by permissions, so ask the user to run it.
5. Two ways to fix a DNS suffix problem (present both, recommend (a)):
   a. Use the FQDN (`<host>.corp.local`) in the connection string / ConfigMap. Precise, one app. A secret value change is done by the user in KV, not by the agent.
   b. Add the suffix to `dnsConfig.searches` in the deployment (gitops PR). Affects every lookup in that pod.
6. If DNS resolves but connection fails, it is network: DB firewall allowlist of the cluster egress IP / NAT; NSG or security group; route to on-prem (VPN / ExpressRoute / TGW); port (SQL 1433, Oracle 1521). Apps migrated AKS -> EKS (or hml -> prd) change egress IP: allowlist often missed. Different envs may use different DB servers.
7. If connected but login fails: credential mismatch between KV and DB. Check ExternalSecret synced and pod restarted after rotation; do not read the value.
8. Classify before concluding: DNS / network path / auth / DB down. Give the one owner to act (app team, DB team, network team, devops PR).

## KEDA / scaling

`kubectl get scaledobject,hpa -n <ns>`. One ScaledObject per Deployment (two = two HPAs fighting). prd/stg = CPU+mem min 2 / max 4, no cron.

## Windows container pods

- Start slow: probes need generous `initialDelaySeconds`.
- ENTRYPOINT must be LogMonitor + ServiceMonitor `w3svc`; an overridden entrypoint breaks env var propagation to the app pool.
- Env config comes as `APPSETTING_*` / `CONNSTR_*` via ConfigurationBuilders; a missing key in Secret/ConfigMap means the app reads the committed default.
- IIS errors: EventLog and IIS logs appear in stdout through LogMonitor.

## Ingress / DNS records

HTTPRoute/Gateway (Envoy) in gitops; CNAME targets per env/cloud are in `app-onboarding.md`. Creating DNS records is not this agent's job (production always asks).

## Good evidence habits

- Quote the key log line, not the log.
- Compare git desired state with live state (`kubectl get -o yaml` of non-secret kinds only).
- Note timestamps; correlate with the deploy bump commit and ArgoCD last sync.
