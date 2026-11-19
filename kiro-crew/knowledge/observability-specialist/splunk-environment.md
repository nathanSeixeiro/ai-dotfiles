# Splunk environment

Splunk is the primary observability platform. Two products:
- **Splunk Enterprise/Cloud (logs, SPL)**: MCP tools `splunk_*`.
- **Splunk Observability Cloud (O11y: APM, infra metrics, detectors)**: MCP tools `o11y_*`.

## Splunk indexes (read from `splunk_get_indexes`, 2026-11)

One index per cluster. Pick the index from the cluster, never search `index=*`.

| Cluster | Index |
|---|---|
| aks-dvp | `ix_dvp_aks` |
| aks-hml (also QA) | `ix_hml_aks` |
| aks-prd | `ix_prd_aks` |
| eks-dev | `ix_dvp_eks` |
| eks-hml (also QA) | `ix_hml_eks` |
| eks-prod | `ix_prd_eks` |

Other: `windows_ad` (AD/Windows events), `main`, `summary`, `history`. Event counts returned by the index API were 0 at read time: do not trust them, run a real query to see data.

## Not yet confirmed (discover, then tell the user to update this file)

- sourcetypes per index: `splunk_get_sourcetypes`
- field names of k8s logs. Typical for splunk-otel-collector: `k8s.namespace.name`, `k8s.pod.name`, `k8s.container.name`, `k8s.cluster.name`. Verify with `index=<ix> earliest=-15m | head 5` and `| fieldsummary`.
- APM environment names (`deployment.environment`): `o11y_get_apm_environments` (call with `environment_name` if the user gave one, otherwise it returns a partial list and you must say it is partial).
- existing detectors/dashboards/saved searches: `o11y_search_alerts_or_incidents`, `splunk_list_alerts`, `splunk_get_knowledge_objects`.
- retention, license/ingest limits, who owns Splunk admin: ask the user.

## Instrumentation standard (.NET apps on Kubernetes)

Source: gitops `docs/reports/otel-instrumentacao-aks-prd*.md`, EKS prd already done.

- Auto-instrumentation by OTel Operator. **Only .NET 8+ on Linux.** .NET < 8, .NET Framework 4.8, Node/static front: not eligible via auto-injection.
- Pod template annotations (in the **deployment**, not the ConfigMap):
  ```yaml
  instrumentation.opentelemetry.io/inject-dotnet: "splunk-o11y/splunk-otel-collector"
  instrumentation.opentelemetry.io/otel-dotnet-auto-runtime: "linux-x64"
  ```
- Container env:
  ```yaml
  - name: SPLUNK_OTEL_AGENT
    valueFrom: { fieldRef: { fieldPath: status.hostIP } }
  - name: OTEL_EXPORTER_OTLP_ENDPOINT
    value: "http://$(SPLUNK_OTEL_AGENT):4318"
  - name: OTEL_EXPORTER_OTLP_PROTOCOL
    value: "http/protobuf"
  ```
  Manual SDK variant seen in EKS QA: `OpenTelemetry__OtlpEndpoint=http://$(SPLUNK_OTEL_AGENT):4317` (gRPC) plus `K8S_POD_NAME` / `K8S_NAMESPACE` via Downward API for correlation.
- The annotation value is `<namespace>/<Instrumentation name>` = `splunk-o11y/splunk-otel-collector`. The collector agent (`splunk-otel-collector-agent`) is reached via node hostIP. Confirm both exist per cluster with `kubectl --context <ctx> -n splunk-o11y get instrumentation,ds`.
- Windows containers: do NOT add `otel-dotnet-auto-runtime: linux-x64`.
- **AppDynamics conflict**: apps with the AppDynamics CLR profiler (`CORECLR_PROFILER` in the Dockerfile) cannot also get OTel auto-injection. Decide which to keep first; ask the user. Pre-OTel migration card pattern: `[APP-D] remove AppDynamics agent (pre-OTel migration)`.
- `dnsPolicy: None` + corporate DNS breaks `.svc` names: that is why hostIP is used. With `ClusterFirst` a Service name would also work. Check each deployment.
- Rollout order: no-caveat apps first, then AppDynamics apps, then Windows-validation apps. One PR per namespace/app group via gitops, labels per work-rules.

## Platform (non-Splunk) stack, as documented in repos

- AKS (Terraform module `observability`): Prometheus (kube-prometheus-stack), Grafana, Loki, Tempo, Alloy. Standards: gitops `.github/instructions/observability-stack.instructions.md`.
- EKS: Terraform modules `observability` and `cloudwatch-observability`; CloudWatch MCP available for EKS logs/metrics.
- Rule of thumb: app logs/traces/APM -> Splunk. Cluster/platform metrics and CloudWatch-native AWS metrics -> Grafana stack / CloudWatch. If the user's question is ambiguous, start in Splunk.
- Local exec reality: dev/hml KEDA cron scales to 0 at night (min 0/max 1, on 7am off 9pm). Silence in Splunk at night in dev/hml/QA is by design, not an outage. Check the ScaledObject first.

## Related

- Runbooks: `splunk-runbooks.md`
- Cluster facts: `~/.kiro/knowledge/k8s-troubleshooter/environment-map.md`
