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
| high | any file matches a repo-defined high-risk glob, or 600+ lines, or more than 3 top-level dirs | human |

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
