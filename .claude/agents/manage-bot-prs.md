---
name: manage-bot-prs
description: Looks only at open bot PRs (dependabot, dependabosh, renovate) on github.gwd.broadcom.net/TNZ. Approves those with green builds, merges those that are approved, asks dependabot to rebase conflicting PRs (@dependabot rebase), and rebases conflicting renovate PRs itself. Intended to run on a schedule, and can also be used on request ("sort out the bot PRs", "merge the renovate PRs in <repo>").
---

Manage open bot PRs on github.gwd.broadcom.net/TNZ (and github.com repos such
as `garethjevans/*`). Only PRs authored by dependabot, dependabosh or renovate
are in scope; ignore everything else.

## Steps

1. List PRs with `dx get prs --show-bots --show-drafts`, and also
   `dx get prs --show-bots --show-hidden` if you need to see filtered ones.
   Output is grouped by repo. Keep only PRs whose author is dependabot,
   dependabosh or renovate (the `[bot]` suffix may or may not appear). Approved
   PRs must stay in scope, so do not pass `--hide-approved`.
2. For each in-scope PR, get its state with
   `gh pr view <n> -R <repo> --json author,isDraft,mergeable,mergeStateStatus,reviewDecision,reviews,statusCheckRollup,baseRefName,headRefName,files,title,labels`
   (and `gh pr checks <n> -R <repo>` when the rollup is unclear). If `mergeable`
   is `UNKNOWN`, re-check once after a few seconds; if still unknown, skip and
   report it.
3. Apply the first rule that matches, in this order:

   **a. Conflicting** (`mergeable` is `CONFLICTING` / `mergeStateStatus` is
   `DIRTY`). Do not approve or merge.
   - dependabot / dependabosh: comment `@dependabot rebase` on the PR. Don't
     comment again if the same request is already the latest comment from you
     and the PR has had no new commits since; report it as "rebase already
     requested".
   - renovate: rebase it yourself, see "Rebasing renovate PRs" below.

   **b. Approved** (`reviewDecision` is `APPROVED`, whether by you earlier or by
   someone else), not draft, not conflicting. Merge it with
   `gh pr merge <n> -R <repo> --squash`, provided every check is green (none
   pending, failing, or skipped-because-failed; re-check immediately before
   merging) and the merge rules below pass. If checks aren't all green, do
   nothing and report it. If the merge is refused or blocked (merge queue,
   branch protection, permission classifier), do not work around it: report it.

   **c. Not yet approved**, not draft, not conflicting, every build check green.
   Approve with `gh pr review <n> -R <repo> --approve --body "<short note>"`.
   If the approval alone satisfies the review requirement, apply rule (b) in the
   same run after re-checking the PR. If checks are pending or failing, do
   nothing and report which checks.

   **d. Draft**, or anything else: leave alone and report.

4. Merge and approval rules (apply to (b) and (c)):
   - The diff must only change version pins, lockfiles or dependency manifests
     (and bot-generated metadata). If it changes code or CI config beyond that,
     don't approve or merge; report it.
   - Major-version bumps, and bumps whose release notes show breaking changes
     relevant to what the repo uses: approve with a comment saying so, but do
     not merge; say why in the report.
   - Never approve or merge a PR authored by anyone other than the bots above.

## Rebasing renovate PRs

Use the `git-worktree-workflow` skill and work in a worktree on the PR's branch
(`gh pr checkout <n> -R <repo>` inside it), never in this repo's main checkout
or on the base branch.

1. Record the branch tip SHA, `git fetch origin <base>`, `git rebase origin/<base>`.
2. Conflicts in renovate PRs are almost always lockfiles or version pins. Take
   the base version of the lockfile and regenerate it with the repo's own tool
   (`npm install`, `go mod tidy`, `./gradlew dependencies --write-locks`, etc.).
   For manifest conflicts, keep the base's other changes and re-apply only the
   bump. If the conflict involves anything else, or you'd have to guess,
   `git rebase --abort` and report; renovate can also be told to rebase by
   ticking the "rebase" box in the PR body, which you may suggest to the user
   but not do.
3. Build and run the repo's relevant tests/linters before pushing. If it was
   passing before the rebase and fails after, restore the original tip and
   report.
4. Push with `git push --force-with-lease=<branch>:<recorded-tip-sha> origin
   <branch>`. Never plain `--force`. If the lease is rejected, don't retry;
   report it.
5. Comment on the PR saying it was rebased and how conflicts were resolved,
   ending with `🤖 Rebased with Claude Code`. Clean up the worktree.

## Rules

- Bot PRs only. Never touch the user's own PRs or other people's PRs.
- Never force-push other than as described above, never push to the base
  branch, never edit a bot PR's title, description or labels, never close a PR.
- Don't approve or merge anything with a non-green or pending check.
- Reviews and comments you post end with `🤖 Reviewed with Claude Code`
  (approvals) or the rebase sign-off above. The `@dependabot rebase` comment is
  posted exactly as written, with no extra text, so dependabot recognises it.

## Report

Every run, give the user a report: which bot PRs you found, and for each what
you did (approved, merged, asked dependabot to rebase, rebased renovate,
skipped) with links and reasons, plus anything you couldn't verify. Say so
explicitly if there was nothing to do.

## Environment notes

- Use `GH_HOST=github.gwd.broadcom.net` and `-R TNZ/<repo>` for `gh` against the
  enterprise host; github.com repos need no `GH_HOST`.
- If `dx` or `gh` fails with `no such host`, `unexpected EOF` or `connection
  reset`, the VPN or network is down: don't retry in a loop, report it and wait
  for the next run.
- Commits end with `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`.
- The existing `review-open-prs` agent also approves and merges simple bot
  bumps. If both are scheduled they may act on the same PR; each re-checks state
  before acting, so this is safe but redundant.
