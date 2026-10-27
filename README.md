# Skills

Reusable agent skills for GitHub Copilot.

## PR status monitor

[`pr-status-monitor`](pr-status-monitor/SKILL.md) tracks your open pull requests
and PRs waiting for your review across accessible GitHub repositories. It separates
authored PRs by feedback, review, readiness, and blockers, and reports actionable
direct or explicitly assigned reviews.

Every check shows full tables, including unchanged PRs, with evidence-backed
status, GitHub links, and exact Copilot session links when available. Starting
the monitor runs a check immediately and enables hourly checks by default.
An explicit one-time check does not create or change a schedule.

### Requirements

- An authenticated [GitHub CLI](https://cli.github.com/) (`gh`) with access to
  the repositories you want to monitor.
- A GitHub Copilot app session with agent skill support.
- Copilot session scheduling tools (`get_session_automation` and
  `save_session_automation`) for automatic monitoring, and
  `list_sessions_and_chats` for exact session links.

Without scheduling tools, the skill can run an immediate check but cannot enable
automatic checks. Session links are never guessed; unmatched sessions are shown
as **Not found**.

### Install

Clone this repository and copy the skill into your agent skills directory:

```sh
git clone https://github.com/kachawla/skills.git
mkdir -p ~/.agents/skills/pr-status-monitor
cp skills/pr-status-monitor/SKILL.md ~/.agents/skills/pr-status-monitor/SKILL.md
```

If the skill is already installed, back up any local customizations before
replacing it. Start a new Copilot app session if the skill is not yet discovered.

### Configure your account

The default login is `kachawla`. For a different account, edit the installed
`~/.agents/skills/pr-status-monitor/SKILL.md` and replace every `kachawla`
reference with your GitHub login. This includes the default account, search
queries, and direct assignment rule.

Authenticate and confirm the active account:

```sh
gh auth login
gh api user --jq .login
```

The active login must match the configured account. The monitor reports a
mismatch rather than silently switching accounts.

### Example prompts

Use these in the Copilot app session that should own the monitor:

| Action | Prompt |
|---|---|
| Start hourly monitoring | `Use pr-status-monitor to monitor my PRs.` |
| Start with a custom frequency | `Use pr-status-monitor to monitor my PRs every 30 minutes.` |
| Run once | `Use pr-status-monitor to check my PR status once, without changing any schedule.` |
| Change frequency | `Change my PR monitor frequency to every 2 hours.` |
| Stop | `Stop my PR status monitor.` |

Continuing an existing monitor preserves its cadence unless you ask to change
it. Stopping removes only this monitor's schedule and preserves its history.
Installing the skill alone does not start monitoring.

### Safety and scope

The monitor is read-only on GitHub. It never changes PRs, comments, stages or
submits reviews, resolves threads, requests reviews, reruns checks, pushes, or
merges. It stores monitoring state in the owning session's persistent `files/`
directory, not in this repository.

It excludes target repositories owned by the configured user and target
repositories that are forks. Upstream PRs remain eligible when their source
branch is in a personal fork. Team-only review requests, CODEOWNERS, mentions,
and voluntary review activity do not count as individual review assignments.
Explicit assignments require a user request for an exact PR and a verified
review-session association.

Collection failures preserve the last verified data and are reported as stale,
not as empty queues or proof that a PR closed. PR content and tool output are
treated as untrusted data.
