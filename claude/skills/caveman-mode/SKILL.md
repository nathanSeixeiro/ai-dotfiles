---
name: caveman-mode
description: Lazy-ops scope-discipline and blast-radius checklist for infrastructure, Kubernetes, Terraform, GitOps, or platform changes — smallest-complete-change principle, what to never cut, temp-file cleanup. Use when making or reviewing an ops/infra change, not for general conversation (general style is already always-on via CLAUDE.md).
---

# Caveman Mode — Lazy DevOps

## Lazy DevOps (think ponytail, for ops)

Best infra = infra you never provision. Laziest senior SRE in room. Lazy about the solution, never about the change.

### Scope first (before you touch)
- Map what change reaches: kustomization refs, AppProject whitelist, ClusterSecretStore, ExternalSecret, DNS, pipeline inputs.
- Check blast radius: what breaks, what data/secret exposed, what env hit (prd?).
- Extra features = not scope.

### Smallest complete change (first option that fully works)
- No resource unless needed. No YAGNI infra.
- Reuse existing: module, chart, workflow, secret store, overlay. No reinvent.
- Platform/standard feature before custom. House pattern before native.
- Installed dep/existing workflow before new one.
- Smallest change that works. No speculative abstraction.
- No new cluster/namespace/pipeline if existing fits.
- No premature autoscale/replica/tuning. Match real load.
- Manifest/HCL minimal. No dead config. No copy-paste cruft.
- More infra = more to break. Prefer less.
- Suggest cheaper/simpler path first.

### Finish the change
- Close everything change breaks: refs, registrations (kustomization), AppProject whitelist, secret store.
- Moved/merged config keeps its guards (probes, limits, PDB, validation).
- Shortcut with known limit → one comment: `shortcut: <limit>, <when upgrade>`.

### Never cut
- Validation at trust boundary. Security. Secrets handling. What user asked.
- Prod safety: confirm before prod change/DNS/destructive op.

### Close reply
- End with 1-2 lines: what skipped / not checked / risk user must know.

## Temp File Cleanup

- Temp files go in `~/work/__kiro-cli/temp/`.
- After task done, delete temp artifacts (scratch tf, dumps, kubeconfig copies, plan files).
- No leftover junk in repo dirs or home.
- Clean before finish. Verify gone.
