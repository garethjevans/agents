---
name: rebase-conflicting-prs
description: Finds the user's own open PRs with merge conflicts (via dx get prs --me), rebases them onto their base branch in a worktree, resolves the conflicts where the resolution is clear, and pushes with --force-with-lease. Intended to run on a schedule, and can also be used on request ("rebase my conflicting PRs", "fix conflicts on <repo> #<n>").
---

Rebase the user's own open PRs that have merge conflicts, on
github.gwd.broadcom.net/TNZ (and github.com repos such as `garethjevans/*`).

## Steps

1. List the user's PRs with `dx get prs --me --show-drafts`. Output is grouped by
   repo. Drafts are included on purpose: conflicts on a draft are still worth
   fixing.
2. Find the PRs with conflicts. The `Mergeable` column is a hint only; confirm
   each candidate with
   `gh pr view <n> -R <repo> --json mergeable,mergeStateStatus,baseRefName,headRefName,headRepositoryOwner,isCrossRepository`
   and keep those where `mergeable` is `CONFLICTING` (or `mergeStateStatus` is
   `DIRTY`). If `mergeable` is `UNKNOWN`, re-check once after a few seconds; if it
   is still unknown, skip and report it.
3. Skip, and report, any PR that:
   - is closed, merged or in a merge queue;
   - is from a fork you can't push to (`isCrossRepository` and no push access);
   - has had a commit pushed in the last 10 minutes (the user may be mid-work);
   - is a bot PR (dependabot, dependabosh, renovate): those can be asked to
     rebase with `@dependabot rebase` by the user, not by this agent.
4. For each remaining PR, use the `git-worktree-workflow` skill to create a
   worktree on the PR's own branch (`gh pr checkout <n> -R <repo>` inside the
   worktree). Never work in this repo's main checkout or on the base branch.
   - `git fetch origin <base>` then `git rebase origin/<base>`.
   - Record the branch tip SHA before rebasing so it can be restored.
5. Resolve conflicts one commit at a time. Resolve only when the correct result
   is clear, for example:
   - lockfiles and generated files: take the base version and regenerate with the
     repo's own tool;
   - both sides added adjacent lines (imports, list entries, changelog, docs,
     test cases): keep both;
   - formatting, renames or moved code where the intent of both sides is obvious;
   - the PR's change is simply superseded by the base (drop that hunk).

   Abort (`git rebase --abort`) and report, leaving the branch untouched, when:
   - the conflict involves logic where the base changed behaviour the PR depends
     on, or the two intents disagree;
   - security, auth, migrations or public API code is involved and the right
     answer isn't obvious;
   - you'd have to guess, or the rebase would take more than a handful of
     conflicted files.
6. After a clean rebase, verify before pushing: build, and run the repo's
   relevant tests and linters (the same ones CI runs, scoped to what changed
   where the repo supports it). If something fails that passed before the rebase,
   try one small fix; if it still fails, restore the original tip and report. If
   it was already failing on the base, say so and carry on.
7. Push with `git push --force-with-lease=<branch>:<recorded-tip-sha> origin
   <branch>`. Never plain `--force`. If the lease is rejected someone pushed in
   the meantime: do not retry, report it.
8. Comment on the PR (`gh pr comment`) saying it was rebased onto `<base>`: which
   files had conflicts, how each was resolved, and what was verified (and what
   wasn't). Flag any resolution the user should double-check. End the comment
   with `🤖 Rebased with Claude Code`.
9. Clean up the worktree when finished.

## Rules

- Only the user's own PRs, only the PR's own branch.
- Never merge, approve, close, or change draft state, title, description or
  labels.
- Never use plain `--force`, never push to the base branch, never rewrite
  commits beyond what the rebase itself does (no squashing, no reword).
- Don't resolve conflicts by blindly taking "ours" or "theirs" for source files.
- Don't dismiss or resolve any review comments.
- If `dx` or `gh` fails with `no such host`, `unexpected EOF` or `connection
  reset`, the VPN or network is down: don't retry in a loop, report it and wait
  for the next run.

## Report

Every run, give the user a report: which PRs you found, which had conflicts,
which you skipped and why, which you rebased (with links and the resolution
summary), which you aborted and why, and anything you couldn't verify. Say so
explicitly if no PRs had conflicts.

## Environment notes

- Use `GH_HOST=github.gwd.broadcom.net` and `-R TNZ/<repo>` for `gh` against the
  enterprise host; github.com repos need no `GH_HOST`.
- Commits end with `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`;
  PR descriptions end with
  `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.
