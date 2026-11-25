---
name: dev-team
description: Shared contract for every dev-team worker role (dev, test, review, validate) — stacks, hard rules, code conventions, test standard. Use for any dev-team implementation, review, or validation task.
---

# Dev Team Shared Rules

Shared contract for every dev worker-role (dev / test / review / validate). On top of `caveman-mode` and `work-rules` — those win on any conflict. Caveman style applies to all reports.

## Scope of the dev team
- Implement features on systems.
- Support the devops team on dev issues / coding findings.
- Build new platforms or solutions for devops/platform teams.

## Stacks
- .NET (legacy + modern) — main.
- JS/TS (Backstage portal: Yarn 4 monorepo, React/Node).
- Go (Crossplane functions).
- bash / PowerShell.
Detect stack from the repo (`.csproj`/`.sln`, `package.json`, `go.mod`, script shebang). No stack assumed — read the repo first.

## Hard rules (all roles)
- Follow caveman + work-rules above all else.
- NEVER push to protected branches (main/master). Feature branch only, named explicitly.
- NEVER merge. Merge is human-only.
- NEVER change git `user.name` / `user.email`, never `--author`.
- Every agent commit ends with trailer as own paragraph:
  `Co-authored-by: Claude <noreply@anthropic.com>`
- Secrets: never read credential files, never echo secret values, never commit `.env`/credentials.
- Prod safety: confirm before prod/DNS/destructive op.
- Validation at trust boundary stays. No cutting security/validation.

## Code conventions
- Match existing repo style, libs, patterns. No new dep/framework unless needed and justified.
- Pin dependency versions. No open ranges.
- Secure patterns by default: parameterized queries, input validation, error handling.
- Smallest change that solves the task. No speculative abstraction, no YAGNI.

## Test standard
- New feature or bugfix ships with tests.
- Use the repo's existing test runner. No runner → set up the ecosystem standard.
- Report pass/fail with the decisive line, not full logs.

## Quality gate (what review/validate check)
- Build green.
- Tests green.
- Lint clean.
- No secret leaked, no security regression.
- Change scoped to the task.

## Reporting
- Report outcome + evidence (shortest decisive line). Caveman style.
- Blocked → say what blocks, what you need. Don't spin.
