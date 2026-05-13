# sync-fork

A [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) skill that installs a GitHub Actions workflow to automatically sync a fork's default branch from its upstream parent on a recurring schedule.

> Trigger when the user explicitly asks to set up automatic fork syncing — e.g.,
> "给这个 fork 加定时同步上游", "set up auto sync from upstream",
> "每 N 小时同步 fork", "auto-pull from the upstream fork".
>
> **Not** for one-off manual syncs (use the GitHub web "Sync fork" button or `gh repo sync`)
> and **not** for syncing branches between non-fork repos.

## What it installs

A single file: `.github/workflows/sync-fork.yml`. It runs on a schedule (default: every 6 hours at minute 17), detects the upstream parent via `gh api repos/$REPO --jq '.parent.full_name'`, and calls `gh repo sync` (which wraps GitHub's `merge-upstream` API — the same as clicking "Sync fork" on the web). Conflicts fail the workflow loudly; clean fast-forwards exit silently.

No commits, no rebases, no force pushes. Pure GitHub-side merges.

## Token strategy: graceful degradation

The workflow uses `${{ secrets.SYNC_FORK_TOKEN || secrets.GITHUB_TOKEN }}` so install is **zero-config by default** and **upgrades cleanly** when needed.

| Upstream behaviour | What works | What's needed |
|---|---|---|
| Never modifies `.github/workflows/**` | Default `GITHUB_TOKEN` | Nothing — just push the workflow file |
| Modifies workflows occasionally | `SYNC_FORK_TOKEN` (PAT with `workflow` scope) | Provision the secret once |
| Mixed (works until it doesn't) | Both — fallback handles it | Add `SYNC_FORK_TOKEN` if/when a sync fails with `workflow scope` error |

Adding the `SYNC_FORK_TOKEN` secret later does **not** require editing the workflow file. The `||` expression picks up the new secret on the next scheduled run.

## Installation

Drop this skill into a Claude Code skills directory:

```bash
git clone https://github.com/zengsipei/sync-fork-skill ~/.claude/skills/sync-fork
```

Then in any fork repo, ask Claude Code: "set up auto sync from upstream every 4 hours" and the skill takes over (pre-flight checks → workflow file → optional PAT → commit & push → verify).

## Repo layout

```
.
├── SKILL.md                  # The skill's instructions for Claude Code
├── templates/
│   └── sync-fork.yml         # The GitHub Actions workflow template
└── README.md                 # This file
```

## Prerequisites

- [GitHub CLI (`gh`)](https://cli.github.com/) — installed and authenticated (`gh auth login`)
- `gh` token needs `repo` scope; if upstream has workflows, also `workflow` scope:
  ```bash
  gh auth refresh -s workflow,repo
  ```

## Manual usage (without Claude Code)

Copy `templates/sync-fork.yml` to `.github/workflows/sync-fork.yml` in your fork. Edit the `BRANCH` env var if your default branch isn't `main`. Commit + push. Done.

If your upstream ever modifies `.github/workflows/**`, add a `SYNC_FORK_TOKEN` repo secret with a PAT carrying `workflow` scope:

```bash
# Skip gh auth refresh if you've already granted workflow scope
gh secret set SYNC_FORK_TOKEN -R <owner>/<repo> --body "$(gh auth token)"
```

## License

MIT — see [LICENSE](LICENSE).
