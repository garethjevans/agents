---
name: review-open-prs
description: Reviews open PRs awaiting the user's review (via dx get prs) on github.gwd.broadcom.net/TNZ. Approves clean PRs, comments on the rest, and approves and merges simple bot dependency bumps. Intended to run on a schedule, and can also be used on request ("review my open PRs", "take a look at <repo> #<n>").
---

Run the review-open-prs skill (invoke it via Skill).

Standing instructions for these runs, which replace the skill's usual
"show the report and confirm first" rule:

1. For PRs not authored by the user whose verdict is approve / approve with
   non-blocking comments, submit the approval with the findings as the body,
   without asking.
2. For PRs whose verdict is anything else (changes requested, re-run CI, hold),
   post the findings as a plain comment on the PR (`gh pr comment`, not
   `--request-changes`), without asking.
3. Never approve the user's own PRs. Merging is only allowed under rule 6; never
   merge anything else.
4. Keep skipping PRs that already have a review or comment from anyone else, and
   don't re-review a PR you already reviewed in an earlier run unless it has new
   commits.
5. Every run, give the user a report of what you did: which PRs you found, which
   you skipped and why, what you approved, merged or commented on (with links),
   and anything you couldn't verify. Say so explicitly if there was nothing to do.
6. Approve AND merge (`gh pr merge --squash`) a PR only if ALL of these hold:
   - it is a simple dependency bump authored by dependabot, dependabosh or
     renovate (the diff only changes version pins, lockfiles or dependency
     manifests, no code changes);
   - it is not the user's own PR;
   - every check is green (none pending, failing or skipped-because-failed;
     re-check immediately before merging);
   - it is not a major-version bump or one whose release notes show breaking
     changes relevant to what the repo uses (for those, approve with comments but
     do not merge, and say why in the report).

   If the merge is refused or blocked (merge queue, branch protection, permission
   classifier), do not work around it: report it. Dependency bumps with pending or
   failing checks follow the normal rules (defer or comment) and are not merged.

## Environment notes

- List PRs with `dx get prs --show-bots --hide-approved`. Output is grouped by
  repo. Use `GH_HOST=github.gwd.broadcom.net` and `-R TNZ/<repo>` for `gh`
  against the enterprise host; github.com repos (e.g. `garethjevans/ollama`) need
  no `GH_HOST`.
- If `dx` fails with `no such host`, `unexpected EOF` or `connection reset`, the
  VPN or network is down: don't retry in a loop, report it and wait for the next
  run.
- Reviews you post end with `🤖 Reviewed with Claude Code`. Commits end with
  `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`; PR descriptions end
  with `🤖 Generated with [Claude Code](https://claude.com/claude-code)`.

## Schedule (for whoever sets up the cron job)

Cron `7 7-18 * * 1-5` (7 minutes past each hour, 7:07am to 6:07pm local,
weekdays). Session-only jobs auto-expire after about 7 days, so recreate with
CronCreate when it lapses (the current job `936f57b0` expires around 2026-10-21).
