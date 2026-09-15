# fleet-ci

Reusable GitHub Actions for one person's fleet of repos. Each project repo
calls these with a few lines and inherits CI, review, risk tiers, and
auto-merge. Change a policy here, tag it, and Dependabot rolls it out.

## Workflows

| Workflow | Trigger in caller | What it does |
|---|---|---|
| `node-ci.yml` | `pull_request`, `push` to main | Lint, typecheck, test, build, optional Playwright. npm or bun. |
| `risk.yml` | `pull_request` | Labels the PR `risk:low` / `risk:medium` / `risk:high`. Arms auto-merge on low. |
| `claude-review.yml` | `pull_request` | Claude Code review that ends with `VERDICT: clean` or `VERDICT: findings`. |
| `review-automerge.yml` | `issue_comment` (created) | Arms auto-merge on `risk:medium` when the verdict is clean. |
| `dependabot-automerge.yml` | `pull_request` | Arms auto-merge on Dependabot patch and minor updates. |

## Risk tiers

| Tier | Rule | Merge |
|---|---|---|
| low | every file matches a low-risk glob and the diff is under 300 lines | auto on green |
| medium | anything not low or high | auto on green plus a clean bot verdict |
| high | anything under `.github/`, any file matching a repo-defined high-risk glob, or 600+ lines, or more than 3 top-level dirs | human |

Auto-merge only completes when the repo's ruleset requires the CI checks, so
"on green" means the checks the ruleset names.

## Minimal caller

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push: { branches: [main] }
  pull_request:
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true
jobs:
  ci:
    uses: JimmayVV/fleet-ci/.github/workflows/node-ci.yml@v1
    with:
      package-manager: bun
      playwright: true
```

```yaml
# .github/workflows/risk.yml
name: Risk
on:
  pull_request:
    types: [opened, synchronize, reopened]
jobs:
  risk:
    uses: JimmayVV/fleet-ci/.github/workflows/risk.yml@v1
    with:
      high-paths: |
        convex/**
        **/billing/**
        .github/workflows/release-*.yml
  dependabot:
    uses: JimmayVV/fleet-ci/.github/workflows/dependabot-automerge.yml@v1
```

```yaml
# .github/workflows/review.yml
name: Review
on:
  pull_request:
    types: [opened, synchronize]
  issue_comment:
    types: [created]
jobs:
  review:
    if: github.event_name == 'pull_request'
    uses: JimmayVV/fleet-ci/.github/workflows/claude-review.yml@v1
    secrets:
      CLAUDE_CODE_OAUTH_TOKEN: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
  arm:
    if: github.event_name == 'issue_comment'
    uses: JimmayVV/fleet-ci/.github/workflows/review-automerge.yml@v1
```

## node-ci inputs worth knowing

| Input | Default | What it does |
|---|---|---|
| `turbo-cache` | `false` | Adds an `actions/cache` step for `.turbo` to every job (checks, test, build, e2e), keyed `turbo-<os>-<sha>` with a `turbo-<os>-` restore prefix, and sets `TURBO_TELEMETRY_DISABLED=1`. Turn it on only when the caller's scripts actually run through Turborepo — the scripts themselves stay whatever the caller's `package.json` says. |

Remote caching via Vercel is optional and not wired up here. If a caller wants
it, Turborepo reads `TURBO_TOKEN` and `TURBO_TEAM` from the environment, and the
caller can pass them through `build-secrets`:

```yaml
jobs:
  ci:
    uses: JimmayVV/fleet-ci/.github/workflows/node-ci.yml@v1
    with:
      turbo-cache: true
    secrets:
      build-secrets: |
        TURBO_TOKEN=${{ secrets.TURBO_TOKEN }}
        TURBO_TEAM=${{ secrets.TURBO_TEAM }}
```

Note that `build-secrets` is exported before the build and e2e steps only, so
remote cache would apply to those two jobs; the local `.turbo` cache above
covers all four.

## Caller permissions

A caller must declare a top-level `permissions:` block at least as wide as
the workflow it calls, or the run fails at startup with no logs. Use:

```yaml
permissions:
  contents: write
  pull-requests: write
  issues: read
  id-token: write
```

for risk.yml, review.yml, and dependabot callers; `contents: read` is enough
for node-ci.yml.

## Repo settings each caller needs

- Allow auto-merge enabled (Settings → General). Private repos need a paid plan.
- A ruleset on the default branch that requires the CI status checks and
  zero approving reviews. `rulesets/default-branch.json` is the template;
  edit the check names to match the caller's job names.
- `CLAUDE_CODE_OAUTH_TOKEN` secret for the review workflow.
- `dependabot.yml` with a `github-actions` ecosystem so the `@v1` tag here
  gets bumped as a low-risk PR.

## Releasing a change here

Edit, commit, then move the tag:

```
git tag -f v1 && git push -f origin v1
```

Callers pinned to `@v1` pick it up on their next run.
