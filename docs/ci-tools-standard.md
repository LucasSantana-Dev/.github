# LucasSantana-Dev CI Tools Standard

Org-wide policy on which CI / code-review / security-scan tools every repo should run, which it should *not* run, and why. The `quality.yml` reusable workflow at `LucasSantana-Dev/.github/.github/workflows/quality.yml` encodes the "should run" portion.

This standard is the result of one decision recorded in detail in the Lucky repo: [`docs/decisions/2026-05-21-replace-plan-limited-review-tools.md`](https://github.com/LucasSantana-Dev/Lucky/blob/main/docs/decisions/2026-05-21-replace-plan-limited-review-tools.md). Read that ADR for the full rationale (cost numbers, alternatives considered, revisit triggers). This file is the org-wide policy that follows from it.

## TL;DR

| Layer | Tool | Status |
|---|---|---|
| **AI code review** | PR-Agent (Codium-ai action, Anthropic-backed) + `claude-review.yml` org reusable | **Required** |
| **SAST** | Semgrep OSS + CodeQL (both via `quality.yml`) | **Required** |
| **Container / IaC / fs scan** | Trivy (via `quality.yml`) | **Required** |
| **Dep CVE pre-merge gate** | OSV-Scanner (via `quality.yml`, default on) | **Required** |
| **Secrets** | GitGuardian GH App | **Required** |
| **Supply chain** | Socket Security GH App | **Required** |
| **Code quality** | SonarCloud GH App | **Required** |
| **Dep updates** | Dependabot (`.github/dependabot.yml`) | **Required** |
| **Workflow + Dockerfile lint** | actionlint + hadolint (via `quality.yml`) | **Required** |
| **Dead code** | knip (via `quality.yml`, opt-in) | Optional |
| **Snyk** | — | **Not recommended** |
| **Greptile** | — | **Not recommended** |
| **CodeRabbit (Pro or Free)** | — | **Not recommended** |

## Why no Snyk / Greptile / CodeRabbit

All three either gate on a free-tier plan that hits caps under normal PR flow, or charge for a slot that the free + Anthropic-backed stack already fills. Detail per tool:

### Snyk

- Free tier: 200 dep tests / month. A single active repo with 5+ PRs/week exhausts this in two weeks.
- Observed in production (Lucky PRs #905, #913, #914, #915, #916 on 2026-05-21): `code/snyk` returns `Code test limit reached` and shows red on every PR for the rest of the cycle.
- Paid tier: ~$25/user/month — overlaps with what Trivy + OSV-Scanner + Socket + Dependabot already cover for $0.
- **Replacement**: `osv-scanner` job in `quality.yml` (Google OSV.dev DB, no plan limits, SARIF output renders identically in the Security tab).

### Greptile

- Free trial only; review body says `Your free trial has ended` once exhausted.
- Paid pricing oriented at teams; no sustainable solo / OSS tier.
- **Replacement**: PR-Agent's `auto_review` + `claude-review.yml`'s narrative review cover both inline-comment and whole-PR review angles.

### CodeRabbit

- Free tier: per-request caps that surface as `Review skipped` on quick-fire PRs.
- Pro tier: same shape with higher caps, still bites on busy days.
- The inline-style nits CodeRabbit specialises in overlap heavily with Semgrep + Sonar + PR-Agent's `auto_improve` (which calls Claude Sonnet 4.6).
- **Replacement**: PR-Agent + Semgrep + Sonar + CodeQL cover this surface without per-request caps.

## What "Required" means in this standard

A repo following this standard:

1. Calls `quality.yml@<ref>` from its own `.github/workflows/quality.yml` (or equivalent name), with `run-osv-scanner: true` (the default).
2. Has the SonarCloud / GitGuardian / Socket Security GH Apps installed.
3. Has `.github/dependabot.yml` configured (per the `2026-05-16-dependabot-batch-handling-policy` ADR — split majors from patches+minors).
4. Runs PR-Agent in `.github/workflows/pr-agent.yml` (Anthropic backend, model `claude-sonnet-4-6` or current default) AND calls `claude-review.yml` from `review-tools.yml`.
5. Has the Snyk, Greptile, CodeRabbit GH Apps **uninstalled** — not just disabled.
6. Has `code/snyk`, `greptile-apps` removed from `required_status_checks` branch protection on `main` and `release/**`.

## Migration plan for a new repo following the standard

1. Add the caller workflow at `.github/workflows/quality.yml` per the template in `docs/quality-workflow.md`.
2. Install SonarCloud / GitGuardian / Socket Security GH Apps. Configure each per their docs.
3. Drop in `.github/workflows/pr-agent.yml` (clone from Lucky repo for the working shape) — needs `ANTHROPIC_API_KEY` secret.
4. Drop in `.github/workflows/review-tools.yml` that calls `LucasSantana-Dev/.github/.github/workflows/claude-review.yml@<ref>`.
5. Configure `.github/dependabot.yml`.
6. Branch protection: pin the `call / *` check names from `quality.yml`'s jobs.

## Migration plan for an existing repo currently on Snyk / Greptile / CodeRabbit

1. Bump `quality.yml@<ref>` to a version that includes `osv-scanner` (this commit forward).
2. Verify the `osv-scanner` job produces clean signal for 1 week (check the Security tab for SARIF findings; suppress known-acceptable transitives via `.osv-scanner.json`).
3. Uninstall Snyk GH App from the org. Remove `code/snyk` from branch protection.
4. Uninstall Greptile GH App.
5. Uninstall CodeRabbit GH App. Cancel the Pro subscription if applicable.
6. Confirm PR-Agent + claude-review still produce review comments. If not, fix wiring before removing the previous tools.
7. CHANGELOG entry: `ci: align with org CI tools standard — drop Snyk/Greptile/CodeRabbit, add OSV-Scanner`.

## Revisit triggers

- **Anthropic API spend on AI review > $50/year for a single repo.** Reassess batching or model tier.
- **OSV-Scanner false-positive rate > 3 alerts/week sustained for >2 weeks.** Suppress-list isn't scaling — consider Trivy fs as the primary dep gate, or accept Snyk's paid tier.
- **PR-Agent (Codium-ai) is abandoned or breaks the action.** Migrate to `anthropics/claude-code-action` (drop-in replacement).
- **A team forms.** CodeRabbit Team or Greptile paid may justify their cost at multi-reviewer scale.
- **GitHub Advanced Security becomes free for private repos.** Likely absorbs the CodeQL + Trivy + OSV signal natively.

## Related decisions

- `2026-05-16-dependabot-batch-handling-policy` (Lucky) — npm dependabot groups split by `update-types`.
- `2026-05-13-base-image-stay-on-alpine` (Lucky) — base-image choice intersects with Trivy CVE findings.
