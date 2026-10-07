# `.github` — org-level GitHub config

Reusable workflows + community health files shared across all `LucasSantana-Dev/*` repos.

## Reusable workflows

### `claude-review.yml`

Self-owned PR reviewer powered by [`anthropics/claude-code-action`](https://github.com/anthropics/claude-code-action). Replaces Greptile (trial cap exhausted) and supplements CodeRabbit. Prompt is tuned to the merge rule in `~/.claude/standards/workflow.md`: only flags correctness, security, semver, prod-risk, and meaningful test gaps. Skips style nits.

Reviews are **on demand only**: a `/claude-review` PR comment from an OWNER, MEMBER or COLLABORATOR triggers one run, capped at `max_reviews` (default 3) per PR. Each run posts a `N/3` marker comment, checks out the PR head, and fails if Claude posts no `Claude review:` summary (no silent green). Fork PRs are refused (the job has secrets).

**Consumer usage:**

```yaml
# .github/workflows/claude-review.yml
name: Claude Review

on:
  issue_comment:
    types: [created]

permissions:
  contents: read
  pull-requests: write
  issues: write
  id-token: write

jobs:
  review:
    name: AI Code Review
    if: >-
      ${{ github.event.issue.pull_request
      && startsWith(github.event.comment.body, '/claude-review')
      && contains(fromJSON('["OWNER","MEMBER","COLLABORATOR"]'), github.event.comment.author_association) }}
    uses: LucasSantana-Dev/.github/.github/workflows/claude-review.yml@<sha>
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

`issue_comment` workflows run from the default branch, so the caller only works once merged there.

Optional inputs: `pr_number`, `max_reviews`, `model`, `max_turns`, `timeout_minutes`, `prompt_override`.

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
| `CLAUDE_CODE_OAUTH_TOKEN` | `claude-review.yml` | `claude setup-token` (Claude subscription), run outside any agent session |
| `ANTHROPIC_API_KEY` | `claude-review.yml` (alternative) | https://console.anthropic.com → API Keys (bills API credits) |
| `GITHUB_TOKEN` | `danger.yml` | auto-provided by GitHub Actions |

The Claude credential must be added to **each consumer repo's** Actions secrets (no org-level secret sync currently — see ADR for the audit cadence).

## Related

- ADR: `LucasSantana-Dev/ai-dev-toolkit:docs/decisions/2026-05-10-multi-repo-review-tools-rollout.md`
- Pilot consumer: `LucasSantana-Dev/Lucky` PR #838 (canonical `dangerfile.ts` reference)
- Merge rule: `~/.claude/standards/workflow.md` "Merge rule" section
