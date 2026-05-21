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
| `osv-scanner` | google/osv-scanner-action (lockfile CVE scan, conditional on `run-osv-scanner`, default true) | Always free, no plan limits |

## Tools NOT in this workflow (configured via GitHub App, per repo — approved)

- **SonarCloud** — install via GH App, add `.sonarcloud.properties`. Free tier OK for solo.
- **GitGuardian** — install via GH App. Free tier covers secret scanning.
- **Socket Security** — install via GH App. Free for OSS + small private.
- **Dependabot** — `dependabot.yml` per repo. Native GitHub.
- **PR-Agent (Codium-ai)** — install via action in each repo's `.github/workflows/pr-agent.yml`, Anthropic-backed. Sole AI code review surface alongside `claude-review.yml` reusable.

## Tools NOT recommended (plan-limited; see `docs/ci-tools-standard.md`)

- **Snyk** — free tier hits "Code test limit reached" on active repos. `osv-scanner` job above replaces the dep-CVE gate at zero cost.
- **Greptile** — trial-only; no sustainable free tier.
- **CodeRabbit** — Pro has per-request caps; free tier identical caps. Redundant with PR-Agent + claude-review.

Why? These tools all gate on plan limits that bite in normal flow. The free-tier stack above plus Anthropic-backed AI review covers the same surface without throttling. Full rationale: `docs/ci-tools-standard.md` and the Lucky ADR it references.

## Caller template

```yaml
# .github/workflows/quality.yml in each repo
name: quality
on:
  pull_request:
  push:
    branches: [main, release]

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
      run-osv-scanner: true       # required by org standard; only disable if repo has no lockfiles to scan
      package-manager: npm        # or pnpm / yarn
      lint-script: lint           # whatever `npm run X` does ESLint+Prettier
```

## Required GH branch ruleset gates

Pin these context names in each repo's `main` AND `release` branch rulesets (`required_status_checks`). See [`branch-strategy.md`](./branch-strategy.md) for the branch model.

- `call / SAST (semgrep)`
- `call / SAST (CodeQL)`
- `call / Vuln + IaC (trivy)`
- `call / Workflow lint (actionlint)`
- `call / Dockerfile lint (hadolint)` (only if `has-dockerfile: true`)
- `call / Lint (<lint-script>)` (only if `lint-script != ''`)
- `call / Dead code (knip)` (only if `run-deadcode: true`)
- `call / Dep CVE (osv-scanner)` (default on; flip `run-osv-scanner: false` to disable)

## Ignored paths

The workflow excludes worktrees, archive, and node_modules by default. Pass
`paths-ignore` if your repo needs additional exclusions.

## Versioning

The workflow is pinned to `@main` in callers. When making breaking changes,
cut a tagged release (`v1`, `v2`) and migrate callers in waves.
