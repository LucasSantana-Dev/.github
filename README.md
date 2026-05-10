# `.github` — org-level GitHub config

Reusable workflows + community health files shared across all `LucasSantana-Dev/*` repos.

## Reusable workflows

### `claude-review.yml`

Self-owned PR reviewer powered by [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action). Replaces Greptile (trial cap exhausted) and supplements CodeRabbit. Prompt is tuned to the merge rule in `~/.claude/standards/workflow.md`: only flags correctness, security, semver, prod-risk, and meaningful test gaps. Skips style nits.

**Consumer usage:**

```yaml
# .github/workflows/review-tools-caller.yml
name: Review Tools

on:
  pull_request:
    types: [opened, synchronize, reopened, ready_for_review]
    branches: [main, 'release/**']
    paths-ignore: ['**.md', 'docs/**', 'CHANGELOG.md', 'package-lock.json']

jobs:
  claude-review:
    uses: LucasSantana-Dev/.github/.github/workflows/claude-review.yml@v1
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

Optional inputs: `model`, `max_turns`, `timeout_minutes`, `prompt_override`.

### `danger.yml`

Runs [Danger.js](https://danger.systems/) against the consumer repo's `dangerfile.ts`. Catches deterministic issues that AI bots silently bail on under rate limits: lockfile drift, `.env` leaks, `console.log` residue, missing CHANGELOG, branch-prefix discipline, big-PR / big-file warnings.

**Consumer usage:**

```yaml
  danger:
    uses: LucasSantana-Dev/.github/.github/workflows/danger.yml@v1
    secrets: inherit
```

Optional inputs: `node_version` (default 22), `danger_version` (default `^12`), `fail_on_errors` (default true), `timeout_minutes`.

The `dangerfile.ts` itself lives at the consumer repo root and is **per-repo by design** — Lucky's TS rules don't apply to a Bash IaC repo. See `ADR 2026-05-10-multi-repo-review-tools-rollout` in `LucasSantana-Dev/ai-dev-toolkit`.

## Versioning

Tag releases as `v1`, `v1.1`, etc. Consumers should pin to a tag (`@v1`), not `@main`, to avoid central breakage taking down all repos at once.

## Required secrets

| Secret | Used by | Where to get |
|--------|---------|--------------|
| `ANTHROPIC_API_KEY` | `claude-review.yml` | https://console.anthropic.com → API Keys |
| `GITHUB_TOKEN` | `danger.yml` | auto-provided by GitHub Actions |

`ANTHROPIC_API_KEY` must be added to **each consumer repo's** Actions secrets (no org-level secret sync currently — see ADR for the audit cadence).

## Related

- ADR: `LucasSantana-Dev/ai-dev-toolkit:docs/decisions/2026-05-10-multi-repo-review-tools-rollout.md`
- Pilot consumer: `LucasSantana-Dev/Lucky` PR #838 (canonical `dangerfile.ts` reference)
- Merge rule: `~/.claude/standards/workflow.md` "Merge rule" section
