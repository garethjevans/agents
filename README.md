# agents

Claude Code subagent definitions for managing pull requests on
`github.gwd.broadcom.net/TNZ` (and `github.com` repos such as `garethjevans/*`).
Each agent is intended to run on a schedule, and can also be invoked on request.

They rely on the `dx` and `gh` CLIs for listing and acting on PRs.

## Layout

```
.claude/agents/
├── manage-bot-prs.md
├── rebase-conflicting-prs.md
├── review-open-prs.md
└── self-review-my-prs.md
```

Each file is a Claude Code agent definition: YAML frontmatter (`name`,
`description`) followed by the agent's instructions.

## Agents

| Agent | Scope | What it does |
| --- | --- | --- |
| `manage-bot-prs` | Open PRs by dependabot, dependabosh or renovate | Approves those with green builds, merges approved ones, asks dependabot to rebase conflicting PRs (`@dependabot rebase`), and rebases conflicting renovate PRs itself. |
| `rebase-conflicting-prs` | Your own open PRs with merge conflicts (`dx get prs --me`) | Rebases them onto their base branch in a worktree, resolves conflicts where the resolution is clear, and pushes with `--force-with-lease`. |
| `review-open-prs` | PRs awaiting your review | Runs the `review-open-prs` skill. Approves clean PRs, comments on the rest, and approves and merges simple bot dependency bumps. |
| `self-review-my-prs` | Your own open, non-draft, non-WIP PRs | Self-reviews those not yet reviewed, fixes easy issues on the PR branch, and comments on the PR to say so. |

## How Claude Code agents work

A subagent is a Markdown file that defines a specialised assistant Claude Code
can delegate to. It runs in its own context window, so its tool calls and file
reads don't fill up the main conversation; only its final report comes back.

### File format

```markdown
---
name: my-agent
description: When and why to use this agent
tools: Bash, Read, Grep      # optional; omit to inherit all tools
model: sonnet                # optional; omit to use the default
---

System prompt / instructions for the agent go here.
```

- `name` identifies the agent (it is the `subagent_type`).
- `description` is how Claude decides when to delegate automatically. Concrete
  trigger phrases help.
- `tools` restricts what the agent can do; omit it to inherit everything.
- `model` lets you run a cheaper or faster model for simple jobs.
- The body is the agent's system prompt. The agent does not see the
  conversation, so the instructions must be self-contained.

### Where agents live

| Location | Scope |
| --- | --- |
| `.claude/agents/` in a project | That project; can be committed and shared |
| `~/.claude/agents/` | All your projects |

If names clash, the project-level agent wins.

### How they are invoked

1. Automatically, when a request matches an agent's `description`.
2. Explicitly, e.g. "use the manage-bot-prs agent".
3. On a schedule, e.g. via `/loop` or a cron-style trigger. The agents in this
   repo are written for this.

### Things to know

- Each run starts fresh. Agents only "remember" things they can read from disk
  or from GitHub (for example, earlier review comments on a PR).
- Subagents can't spawn further subagents.
- Several can run in parallel when the work is independent.
- They run under the session's permission mode, so unattended runs need the
  relevant commands (`gh`, `dx`, `git`) allowed in settings.
- Agent files are loaded at session start; restart or use `/agents` after
  editing. `/agents` also creates and manages agents interactively.
- Skills and agents combine well: `review-open-prs` is a thin agent that runs
  the skill of the same name, adding standing rules for unattended runs.

## Usage

Run from this directory (or any project that picks up these agents), either by
name or in plain language:

- "sort out the bot PRs", "merge the renovate PRs in `<repo>`"
- "rebase my conflicting PRs", "fix conflicts on `<repo>` #`<n>`"
- "review my open PRs", "take a look at `<repo>` #`<n>`"
- "self-review my PRs", "self-review `<repo>` #`<n>`"

Read each agent file for its exact steps and the rules it follows, such as which
PRs it skips and when it is allowed to approve or merge.
