# `quality.yml` — org-wide reusable QA + security workflow

Single source of truth for the free-tier static analysis stack across all
LucasSantana-Dev repos. Callers stay tiny.

## What it runs

| Job | Tool | Free-tier note |
|-----|------|----------------|
| `semgrep` | semgrep CI (p/default + p/security-audit + p/owasp-top-ten) | Free OSS rules; SARIF uploaded to Code Scanning |
| `codeql` | github/codeql-action (security-and-quality) | Free for public repos; SARIF auto-uploaded |
| `trivy` | aquasecurity/trivy-action (fs scan, vuln + IaC + secrets) | Always free; SARIF uploaded |
| `actionlint` | rhysd/actionlint | Always free |
| `hadolint` | hadolint/hadolint-action (conditional on `has-dockerfile`) | Always free |
| `lint` | repo's own `npm run <lint-script>` | Always free |
| `deadcode` | knip via npx (conditional on `run-deadcode`) | Always free |

## Tools NOT in this workflow (configured via GitHub App, per repo)

- **SonarCloud** — install via GH App, add `.sonarcloud.properties`
- **GitGuardian** — install via GH App
- **Snyk** — install via GH App (legacy: workflow-based, but app is easier)
- **Socket Security** — install via GH App
- **CodeRabbit** — install via GH App
- **Dependabot** — `dependabot.yml` per repo

Why? These tools all have their own GH App that does what you want without a
workflow file. Trying to invoke them via GHA duplicates config and breaks the
free-tier rate limits.

## Caller template

```yaml
# .github/workflows/quality.yml in each repo
name: quality
on:
  pull_request:
  push:
    branches: [main, "release/**"]

jobs:
  call:
    uses: LucasSantana-Dev/.github/.github/workflows/quality.yml@main
    permissions:
      contents: read
      security-events: write
      actions: read
    with:
      node-version: "22"
      has-dockerfile: true        # set false for repos without Dockerfiles
      run-deadcode: true          # opt-in: requires knip configured in repo
      package-manager: npm        # or pnpm / yarn
      lint-script: lint           # whatever `npm run X` does ESLint+Prettier
```

## Required GH branch ruleset gates

Pin these context names in each repo's main-branch ruleset (`required_status_checks`):

- `call / SAST (semgrep)`
- `call / SAST (CodeQL)`
- `call / Vuln + IaC (trivy)`
- `call / Workflow lint (actionlint)`
- `call / Dockerfile lint (hadolint)` (only if `has-dockerfile: true`)
- `call / Lint (<lint-script>)` (only if `lint-script != ''`)
- `call / Dead code (knip)` (only if `run-deadcode: true`)

## Ignored paths

The workflow excludes worktrees, archive, and node_modules by default. Pass
`paths-ignore` if your repo needs additional exclusions.

## Versioning

The workflow is pinned to `@main` in callers. When making breaking changes,
cut a tagged release (`v1`, `v2`) and migrate callers in waves.
