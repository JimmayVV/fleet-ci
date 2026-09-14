# Solo Speedrun Playbook

How to run a fleet of solo projects so that shipping is fast and review only
costs human time where it buys something. Distilled from Amplitude's
"Speedrunning Software" (amplitude.com/3x, tripled PRs in six months) and
rewritten for one maintainer who is every code owner. Written 2026-09-14.

## The thesis

Agents made writing code cheap. The pipeline is Ideas → Code → Validation →
Review → Production, and once Code is cheap, Validation and Review are the
constraint. Fix those foundations before building anything clever on top;
Amplitude tried custom agents first and it failed until CI and local dev were
fixed. Every time one bottleneck goes, the next one appears, so measure after
each change, not at the end.

## The six moves, in order

1. **Foundations before agents.** Fast CI and zero-setup validation first.
2. **Sub-five-minute CI.** Build only what changed (Turborepo affected builds,
   cached outputs), swap JS tooling for Rust and Go ports (tsc→tsgo,
   ESLint→oxlint, Prettier→oxfmt, esbuild→rolldown, Jest→Vitest), run
   independent stages in parallel.
3. **Zero-setup validation.** A PR label triggers a preview with deep links
   to the changed pages. Nobody needs a local checkout to verify.
4. **Risk-scored review.** Score size, scope, location, test coverage, and
   public-API or data-model changes. Low risk auto-merges. Medium and high
   route to code owners.
5. **A bot in every review.** Automated review with repo context. Amplitude's
   bug reports fell 55% while volume tripled.
6. **Compliance as logged criteria.** Documented auto-approval rules, an
   escalation path, and a logged decision satisfy audit without a human click
   on every change.

## Translating to one person

You are every code owner, so routing collapses to one question: which changes
deserve your eyes before merge, and which should never wait on you?

| Tier | Rule | Merge |
|---|---|---|
| **low** | every file matches a low-risk glob (docs, tests, formatting, CI config, Dependabot patch/minor) and the diff is under 300 lines | auto on green |
| **medium** | ordinary source changes that are not low or high | auto on green **plus** a clean bot verdict |
| **high** | any file under `.github/`, or any file matches a repo-defined protected glob (schema and migrations, auth and billing, release and signing workflows, public interfaces, FFI boundaries), or 600+ lines, or more than 3 top-level dirs, or a Dependabot major | a human, always |

Standing rule that predates this: a green review job is not the merge bar.
Read the bot's findings and either fold them in or consciously defer with a
reply on the PR. The medium tier automates that rule: the bot's comment must
end in `VERDICT: clean` for auto-merge to arm.

## The implementation

Everything lives in `JimmayVV/fleet-ci` as reusable workflows, and a project
repo calls them in a few lines. Change a policy here, move the `v1` tag, and
every repo inherits it. Dependabot's `github-actions` ecosystem bumps the tag.

- `node-ci.yml` lint, typecheck, test, build, optional Playwright, npm or bun
- `risk.yml` labels `risk:low|medium|high`, arms auto-merge on low
- `claude-review.yml` review ending in a verdict line
- `review-automerge.yml` arms auto-merge on medium when the verdict is clean
- `dependabot-automerge.yml` patch and minor auto-merge, majors flagged high
- `rulesets/default-branch.json` required checks, zero required reviews, squash only

Repo settings each caller needs: allow auto-merge (private repos need a paid
plan on a personal account), a default-branch ruleset requiring the CI checks,
the `CLAUDE_CODE_OAUTH_TOKEN` secret, and `dependabot.yml` with a
`github-actions` entry. New projects start from `JimmayVV/fleet-template`.

## Tech stack baseline

- **TypeScript:** bun, Vite, vitest, Playwright for UI, oxlint, oxfmt, tsgo
  once it handles the repo's tsconfig. One lockfile format across the fleet.
- **Rust:** cargo fmt, clippy with warnings denied, Swatinem/rust-cache.
- **Static sites:** Astro or React Router with one build check. Netlify or
  GitHub Pages. Do not add a third host.
- **Backend:** Convex where a backend is needed; its preview deployments cover
  validation.

## Turborepo, honestly

It works in a single-package repo and the Vercel remote cache is free on every
plan. In that mode you get content-hashed task caching, task ordering with
parallelism, and watch mode. The headline win, affected-package builds, only
exists across a workspace graph. Verdict: yes on day one for real monorepos
(marchland, pylon); a measured trial on single-package Vite apps, kept only
if median CI time drops; JS side only on Tauri apps; no for Rust, Astro,
plain Node, or shell projects.

## What to measure, monthly, per repo

1. PR cycle time, open to merge, median, split by risk label.
2. Share of PRs merged with no human review. Under half means the tiers are
   too conservative.
3. CI wall time, median. The bar is five minutes.
4. Reverts and hotfixes within seven days of an auto-merged PR. The only
   number that says whether auto-merge is safe. If it rises, tighten medium,
   not low.

## Rollout order for a new or existing repo

1. CI via `node-ci.yml`. If the repo never had CI, expect failures; that is
   the point.
2. Risk labels only. Watch a week of PRs and check the labels match instinct.
3. Ruleset with required checks, auto-merge enabled.
4. Review workflow with the verdict line, then medium-tier auto-merge.
5. Dependabot with groups and the actions ecosystem.
6. Turborepo trial, measured.
