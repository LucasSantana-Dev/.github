# LucasSantana-Dev Branch Strategy

Org-wide policy on which branches every repo uses, what PRs target, and how releases get cut. Pairs with [`ci-tools-standard.md`](./ci-tools-standard.md) — that doc covers tools, this doc covers branches.

## TL;DR

| Branch | Role | Who pushes here directly | What lives here |
|---|---|---|---|
| `main` | Released code only | **Nobody.** | Whatever has shipped to production / been tagged |
| `release` | Long-lived integration branch | **Nobody.** PRs target it. | Everything merged since the last release cut |
| `feature/*`, `fix/*`, `chore/*`, `test/*`, `refactor/*`, `docs/*`, `ci/*` | Short-lived work branches | Author | The change in progress |
| `hotfix/*` | Production-degraded emergency only | Per the `hotfix` skill | One narrow fix to a tagged release |

**Default PR base: `release`.** Never `main`. The only time `main` is the base is the `release → main` PR that ships a version (the `/release-cut` workflow handles that).

## Why bare `release`, not versioned `release/vX.Y.Z`

Trunk-based literature describes both shapes. We've chosen the bare-branch variant for the org because:

1. **One long-lived integration target.** PRs aren't ambiguous about which release branch they belong to. The version of "what's currently on release" is determined by the next tag, not by the branch name.
2. **`/pr-to-release` skill detects `release` natively** (`git ls-remote --heads origin release`). No per-repo override needed.
3. **Tags carry the version.** `v2.11.0`, `v2.12.0`, `v2.12.1` are created on `release` by `/release-cut`, then `release` is fast-forwarded into `main`. The branch itself is monotonically forward — no v2.12.0 branch to clean up, no v2.13.0 to cut at start.
4. **Cleanup is mechanical.** After a release ships, the local `release` ref just keeps moving. Versioned branches accumulate as old refs nobody deletes.

## Lifecycle of a change

```
feature/x  ──→  release  ──→  main  ──→  tag vN.M.0
                                          ↓
                                       deploy
```

1. Cut a short-lived branch from `release` (not from `main`). Name it with one of the approved prefixes.
2. Open the PR with **base = `release`**. Default branch in GitHub UI is `main`; you must change it.
3. Land the PR (squash merge) once green. The change accumulates on `release` with all other in-flight work.
4. When the batch is large enough or the cycle deadline arrives, run `/release-cut`:
    - Open the `release → main` PR.
    - Version bump (semver per conventional-commit subjects accumulated on `release`).
    - Promote `CHANGELOG.md` `[Unreleased]` block to a dated version section.
    - Squash-merge to `main`. Tag `vN.M.P` on the merge commit.
5. Deploy from the new tag (per the per-repo deploy workflow, not part of this strategy).

`release` never has stale work — it's always "what would ship if I cut now."

## What this means in practice

- **Don't target `main` in a PR.** Branch protection on `main` is set to allow only the `release → main` merge.
- **Don't commit to `main` locally.** Pulling main is fine; pushing isn't.
- **Don't commit to `release` locally either.** It's a merge target, not a working branch. The only commits on `release` are squashed PR merges from feature branches + the `release → main` PR's commit.
- **No long-lived feature branches.** If something can't ship inside the next release cycle, it's still merged behind a feature toggle. The branch is short-lived; the feature isn't.
- **Hotfix exception.** `/hotfix` cuts from the last release tag, lands directly to `main`, then is back-merged to `release`. The full criteria live in the `release-cadence.md` standard ("Hotfix policy" section).

## Required branch protection (per repo)

On `main`:
- Required pull request before merging
- Allow only the `release` branch as PR source (where supported)
- Require linear history
- Required status checks: all of the `quality.yml` job names (see [`quality-workflow.md`](./quality-workflow.md))

On `release`:
- Required pull request before merging
- Required status checks: same `quality.yml` jobs as `main`
- Allow squash merge only (one PR → one commit on release)

Feature branches: no protection. Force-push allowed (rebase + force-with-lease is fine).

## How this interacts with skills

- **`/pr-to-release`** — the default workflow for landing any change. Detects `release` automatically.
- **`/release-cut`** — the only workflow that creates a tag and updates `main`.
- **`/hotfix`** — the only sanctioned bypass of `release`. Production-degraded or actively-exploited only.
- **`/merge-confidently`** — direct-to-main; should now only fire on repos that explicitly opt out of this strategy (rare).
- **`/ship`** — pairs with `/release-cut`; refuses admin bypasses.

## Migration for existing repos

Repos still using versioned `release/vX.Y.Z` (today: Lucky), or no release branch (most policy/config repos):

### Versioned → bare (Lucky pattern)

1. Confirm there's a clean snapshot tag on the current branch tip (`vN.M.0` on `release/vN.M.0`).
2. `git push origin release/vN.M.0:release` — creates the bare `release` branch at the same point.
3. Migrate any in-flight PRs targeting `release/vN.M.0` to target `release` (`gh pr edit <num> --base release`).
4. Update branch protection: protect `release`, keep the old versioned branch as an archive (or delete after the next release tag confirms parity).
5. Update each caller workflow file that hardcoded `release/**` to also include `release` (or replace).

### No release branch → adopt

1. `gh api -X POST /repos/<owner>/<repo>/git/refs -f ref=refs/heads/release -f sha=<main HEAD SHA>`
2. Set branch protection on `release` per the rules above.
3. Update any in-flight PRs targeting `main` to target `release`.
4. Document the policy in the repo's CLAUDE.md / README.

## Revisit triggers

- **A repo grows enough that one `release` branch becomes a bottleneck** (multiple teams merging concurrently with conflicting work). Reconsider versioned branches at that point — not before.
- **The "no commits on `release`" invariant gets broken accidentally more than once.** May indicate the protection rules are too permissive or the team needs a workflow refresher.
- **`/pr-to-release` skill changes its detection logic.** Re-check the docs.

## Related artefacts

- [`ci-tools-standard.md`](./ci-tools-standard.md) — what runs against each PR
- [`quality-workflow.md`](./quality-workflow.md) — how the reusable QA workflow plugs in
- `~/.agents/skills/standards/release-cadence.md` — semver + hotfix policy detail
- `~/.agents/skills/standards/pr-conventions.md` — PR title / body conventions
