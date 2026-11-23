---
paths:
  - ".github/workflows/**"
  - "**/.github/workflows/**"
---

# GitHub Actions Rules

<!-- CI/CD-specific enactment of the general security baseline in rules-security.md
     (secrets management, least privilege) — kept here because the specifics
     (OIDC, GITHUB_TOKEN scoping, SHA pinning) don't generalize. -->

## Workflow Structure
- One workflow per concern (ci.yml, deploy.yml, release.yml) rather than one monolithic pipeline
- Split long workflows into reusable workflows (`workflow_call`) or composite actions to avoid duplication across files
- Name jobs and steps descriptively; the Actions UI is the primary debugging surface
- Use `job.<id>.needs` to express real dependencies; avoid serializing independent jobs just for readability

## Secrets & Permissions
- Default `permissions: {}` at the workflow level, then grant only what each job needs (least-privilege `GITHUB_TOKEN`)
- Prefer OIDC (`permissions: id-token: write`) over long-lived cloud credentials for deploys to AWS/Azure/GCP
- Never echo secrets to logs; mask custom secret-like values with `::add-mask::` if constructed at runtime
- Store secrets in the repo/org secret store, not in workflow YAML or committed `.env` files — see rules-security.md for the general secret-management principle
- Scope secrets to the minimum environment (repository vs. environment-level secrets) that needs them

## Supply Chain Security
- Pin third-party actions to a full commit SHA, not a mutable tag (`uses: actions/checkout@<sha> # v4.1.1`)
- Enable Dependabot for GitHub Actions version updates so pinned SHAs stay current
- Avoid `pull_request_target` with untrusted checkout + code execution — it grants secrets access to fork PRs
- Restrict workflows that can approve/merge/deploy to `pull_request` (not `pull_request_target`) unless secrets are genuinely required

## Caching & Performance
- Cache dependency directories (`~/.npm`, `~/.cache/pip`, `~/.m2`) keyed on the lockfile hash
- Use `actions/setup-*` built-in caching (`cache: 'npm'`, etc.) before reaching for manual `actions/cache`
- Use a matrix build (`strategy.matrix`) for multi-version/multi-OS testing instead of duplicated jobs
- Set `fail-fast: false` on matrices where one version failing shouldn't cancel the others' diagnostic value
- Set explicit `timeout-minutes` on every job to prevent runaway/hung runs from burning CI minutes

## Reliability
- Fail the workflow loudly on lint/type/test errors — no `continue-on-error: true` on required checks
- Use concurrency groups (`concurrency: { group: ..., cancel-in-progress: true }`) to cancel superseded runs on the same ref
- Validate workflow YAML with `actionlint` in CI before merging changes to `.github/workflows/`
