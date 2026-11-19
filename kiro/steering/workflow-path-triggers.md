---
inclusion: always
name: "Workflow Path Triggers"
---

# Workflow Path Triggers

Applies to every CI/CD workflow created or changed for repos that hold several apps (e.g. `multi-app-web`: AppA, AppB, SiteA, SiteB).

## Rule
- One workflow per app. Its `on.push.paths` lists only what changes that app's image:
  - the app's own project folder (`<App>/**`)
  - its own Dockerfile (`docker/Dockerfile.<App>`), never `docker/**`
  - its own workflow file (`.github/workflows/<app>.yml`)
- Shared paths are allowed only when the app really depends on them:
  - `packages/**` (shared NuGet folder)
  - `Business/**` or another library, only if the csproj has a `ProjectReference` to it
  - shared docker files used by the app (e.g. `docker/LogMonitorConfig.json`)
- A change that touches only app A must not run the workflow of app B.
- Never use catch-all globs (`docker/**`, `**`, `.github/**`) in `paths`.

## Check before opening a PR
- For each workflow: every `paths` entry is the app's own, or shared with a reason (ProjectReference / shared file).
- Compare with the sibling workflows: no entry of another app's folder, Dockerfile or workflow file.
- `workflow_dispatch` stays enabled so a run can be forced when a change falls outside `paths` (e.g. `.dockerignore`).

## Known limit
`.dockerignore` and root files are not in `paths`, so changing them does not trigger a run. Use `workflow_dispatch`.
