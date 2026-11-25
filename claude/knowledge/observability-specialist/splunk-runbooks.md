# Splunk runbooks and query patterns

## Ground rules

- Always bound the query: `index=<ix_...>` + `earliest=` (start at `-15m`, widen only if empty) + narrow filter before any `stats`.
- Prefer `tstats`/metrics when the data is indexed fields. Avoid `join`, avoid leading wildcards, avoid `transaction` on wide windows.
- `splunk_run_query` returns rows to the chat: add `| head 50` or an aggregation. Never dump raw logs.
- Logs can hold tokens, connection strings, personal data (CPF, e-mail). Quote only needed lines, mask values.
- Read-only. Creating/changing detectors, saved searches, dashboards goes through a PR or a user-run action, never ad hoc.

## SPL recipes (verify field names first, see splunk-environment.md)

Is the app logging at all:
```
index=ix_prd_aks earliest=-15m k8s.namespace.name="ns-corp-cee-prd" | stats count by k8s.pod.name
```
Error volume per pod, last hour:
```
index=ix_prd_aks earliest=-1h k8s.namespace.name="<ns>" (ERROR OR Exception OR " 500 ")
| timechart span=5m count by k8s.pod.name
```
Top error messages:
```
index=ix_prd_aks earliest=-1h k8s.namespace.name="<ns>" ERROR
| rex field=_raw "(?<err>Exception[^\n]{0,120})" | stats count by err | sort -count | head 20
```
Restart/OOM evidence (events forwarded): search `OOMKilled`, `Back-off restarting`, `Liveness probe failed` in the cluster index.
Volume per namespace (cost/noise):
```
| tstats count where index=ix_hml_aks earliest=-24h by host | sort -count | head 20
```

## APM triage flow (O11y MCP, in this order)

1. `o11y_get_apm_environments` -> pick the env (e.g. the prd one).
2. `o11y_get_apm_services_summary` / `o11y_get_apm_services` -> is the service reporting? Not listed = not instrumented or collector path broken.
3. `o11y_get_apm_service_errors_and_requests(_summary)` -> error rate and request rate, compare with a prior window.
4. `o11y_get_apm_service_latency(_summary)` -> p50/p90/p99. A p99-only rise = tail (DB lock, cold start, GC).
5. `o11y_get_apm_service_dependencies` -> downstream (SQL, HTTP) that explains it.
6. `o11y_get_apm_exemplar_traces(_summary)` then `o11y_get_apm_trace_tool` / `o11y_get_apm_trace_summary` -> slow/error trace, find the span.
7. Correlate with Splunk logs by trace id / pod / time window.

Metrics: `o11y_get_metric_names`, `o11y_get_metric_metadata`, then `o11y_generate_signalflow_program` + `o11y_execute_signalflow_program` for ad hoc charts. Alerts: `o11y_search_alerts_or_incidents`.

## Symptom -> where to look

| Symptom | First signal | Then |
|---|---|---|
| 5xx spike | APM errors by endpoint | exemplar trace, app logs same window, last deploy bump in gitops (`chore(<app>): bump`) |
| Slow | APM latency p99 + dependencies | DB span, pod CPU throttling (infra metrics), KEDA scale state |
| Pod restarts | k8s events + logs `--previous` (k8s-troubleshooter) | OOM: memory metric vs limit |
| No data in Splunk | collector pods in `splunk-o11y`, hostIP:4318 reachable, annotation/env present in deployment | .NET version eligible? AppDynamics conflict? Windows image? |
| Missing traces only | `OTEL_EXPORTER_OTLP_*` env, runtime annotation, Linux image | sampling, instrumentation CR image pull |
| Silent at night (dev/hml/QA) | ScaledObject cron | by design |

For cluster-level diagnosis hand off to `k8s-troubleshooter`; you own the telemetry side.

## Detector / alert standards (propose, user approves)

- Golden signals: error rate, latency (p99), traffic, saturation. Per service, per env.
- Severity: Critical = user-facing outage prd (page). Major = degraded. Minor/Info = hml/dev, no page.
- Condition has a duration (`for 5m`) to avoid flapping. Thresholds relative to baseline, not guesses: query 7 days first.
- Every detector: owner team, runbook link, env tag, description with the first 3 checks.
- dev/hml/QA are scaled to zero at night: mute or schedule those detectors, otherwise they fire every evening.
- No detector without a way to test it (replay on historical data in the chart).
- Alert/notification routing is not yours to change: propose, user applies.

## SLO sketch (when asked)

SLI = good requests / total (non-5xx and < threshold ms). Start with 99.5% / 30d for prd apps, error budget burn alerts at 2% in 1h (fast) and 5% in 6h (slow). Adjust with the app owner.

## Rollout of OTel instrumentation (gitops PRs)

1. Classify the app: .NET >= 8, Linux, AppDynamics yes/no (read Dockerfile + `.csproj`, not port heuristics).
2. One PR per namespace group. Edit deployment (annotations + env), not ConfigMap.
3. Labels per work-rules (cluster + type). Branch from up-to-date main.
4. After merge + ArgoCD sync: confirm rollout, then confirm the service shows in APM (`o11y_get_apm_services`) and logs have trace correlation. A change is not done until data appears.
5. prd: hand the sync/verification command to the user, do not mutate.
