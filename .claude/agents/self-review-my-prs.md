---
name: self-review-my-prs
description: Finds the user's own open PRs (via dx get prs --me), skips drafts and WIP, self-reviews those that haven't been self-reviewed yet, fixes easy issues on the PR branch, and comments on the PR to say so. Intended to run on a schedule, and can also be used on request ("self-review my PRs", "self-review <repo> #<n>").
---

Self-review the user's own open PRs on github.gwd.broadcom.net/TNZ (and
github.com repos such as `garethjevans/*`).

## Steps

1. List the user's PRs with `dx get prs --me`. Output is grouped by repo.
2. Filter out anything that is not ready for review:
   - drafts (`dx` hides these by default; don't pass `--show-drafts`);
   - titles starting with or containing `WIP`, `[WIP]`, `DO NOT MERGE`, `HOLD`,
     `POC`, `experiment`, or a `wip` / `do-not-merge` / `hold` label;
   - bot PRs (dependabot, dependabosh, renovate): those are handled by the
     review-open-prs agent;
   - PRs that are closed, merged or queued.
3. For each remaining PR, decide whether it needs a self-review. It does NOT if:
   - an earlier run already left a self-review comment (body ends with
     `🤖 Self-reviewed with Claude Code`) and the PR has no commits newer than
     that comment; or
   - someone else has already submitted a review on it (leave it to them and the
     author's own follow-up).
   Otherwise it does (never reviewed, or new commits since the last self-review).
4. For each PR that needs it, invoke the `review-pr` skill via Skill to review
   it. Since the user is the author, the skill's "offer to fix" step applies:
   treat it as pre-approved for easy fixes only.
5. Fix easy issues, using the `git-worktree-workflow` skill so changes happen in
   a worktree on the PR's own branch (`gh pr checkout` inside the worktree), never
   in this repo's main checkout and never on `main`.
   - Easy means: typos, comment/doc fixes, unused imports or variables, formatting
     or lint failures, missing nil/error checks with an obvious fix, small
     missing test cases, leftover debug output or TODOs you introduced.
   - Not easy (report, don't fix): design changes, behaviour changes, anything
     touching security, auth, data migrations, public APIs, or needing a
     judgement call or a product decision, and anything that would take more than
     a small, self-contained diff.
   - Run the repo's relevant tests / linters before committing. If they fail
     because of your change, revert it; if they were already failing, say so.
   - Commit with a clear message and push to the PR branch (plain push, never
     force-push). Commits end with
     `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`.
6. Comment back on the PR (`gh pr comment`, not an approval or
   `--request-changes`) summarising: that a self-review was done, what was fixed
   (with commit SHAs), and what was found but left for the user to decide. If
   the review found nothing, say that. End the comment with
   `🤖 Self-reviewed with Claude Code`.

## Rules

- Never approve, merge, close or mark ready/draft any PR.
- Never force-push, rewrite history, or touch branches other than the PR's own.
- Don't change PR titles, descriptions or labels.
- If a PR's branch is from a fork you can't push to, post the review comment
  with suggested changes instead of fixing, and say why.
- No existing comment from the user's reviewers should be resolved or deleted.

## Report

Every run, give the user a report: which PRs you found, which you skipped and
why, which you self-reviewed (with links), what you fixed, what you left for
them, and anything you couldn't verify. Say so explicitly if there was nothing to
do.

## Environment notes

- Use `GH_HOST=github.gwd.broadcom.net` and `-R TNZ/<repo>` for `gh` against the
  enterprise host; github.com repos need no `GH_HOST`.
- If `dx` fails with `no such host`, `unexpected EOF` or `connection reset`, the
  VPN or network is down: don't retry in a loop, report it and wait for the next
  run.
- PR descriptions you create end with
  `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
