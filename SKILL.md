---
name: sync-fork
description: Install a GitHub Actions workflow that automatically syncs a fork's default branch from its upstream parent on a recurring schedule. Use when the user explicitly asks to set up automatic fork syncing — e.g., "给这个 fork 加定时同步上游", "set up auto sync from upstream", "每 N 小时同步 fork", "auto-pull from the upstream fork". Do NOT use for one-off manual syncs (the user can click "Sync fork" on GitHub or run `gh repo sync` themselves) or for syncing branches between non-fork repos.
---

## What this skill installs

A single file: `.github/workflows/sync-fork.yml`. It runs on a schedule (default: every 6 hours), detects the upstream parent via `gh api repos/$REPO --jq '.parent.full_name'`, and calls `gh repo sync` (which wraps GitHub's `merge-upstream` API — the same as clicking "Sync fork" on the web). Conflicts fail the workflow loudly; clean fast-forwards exit silently.

That's it. No commits, no rebases, no force pushes. Pure GitHub-side merges.

## When to use this skill

✅ User wants **set-and-forget** auto-sync for a fork they don't actively develop on (e.g., tracking upstream Codex / vendored libs / mirrored docs).

✅ User says any of: "set up auto sync", "定时同步 fork", "每 N 小时同步上游", "automate the sync fork button", "schedule fork sync".

❌ User wants a **one-time** sync — just run `gh repo sync <repo> --branch <branch>` directly or click the GitHub web button.

❌ User wants to sync a non-fork (e.g., mirror two independent repos) — this skill doesn't apply; the workflow refuses to run when `gh api repos/$REPO --jq '.parent.full_name'` returns empty.

❌ User actively develops on the fork's default branch and the upstream may diverge — sync conflicts will fail the workflow on every run; better to use a separate tracking branch or a manual cadence.

## Token strategy: graceful degradation

The workflow uses `${{ secrets.SYNC_FORK_TOKEN || secrets.GITHUB_TOKEN }}` so the install path is **zero-config by default** and **upgrades cleanly** when needed.

| Upstream behaviour | What works | What's needed |
|---|---|---|
| Never modifies `.github/workflows/**` | Default `GITHUB_TOKEN` | Nothing — just push the workflow file |
| Modifies workflows occasionally | `SYNC_FORK_TOKEN` (PAT with `workflow` scope) | Provision the secret once |
| Mixed (works until it doesn't) | Both — fallback handles it | Add `SYNC_FORK_TOKEN` if/when a sync fails with `workflow scope` error |

**Key property**: adding the `SYNC_FORK_TOKEN` secret later does NOT require editing the workflow file. The `||` expression picks up the new secret on the next scheduled run.

The default `GITHUB_TOKEN` is forbidden from merging upstream commits that touch `.github/workflows/**` (it lacks the `workflow` scope). Failure mode is:

```
Upstream commits contain workflow changes, which require the
`workflow` scope or permission to merge.
```

## Pre-flight checks (do these BEFORE writing the file)

Run all of these in parallel — they're cheap and catch the common failure modes:

1. **Confirm we're at the repo root** — `git rev-parse --show-toplevel` should match cwd.

2. **Resolve the GitHub repo identity.** The remote URL might be a custom proxy (e.g. `xget.example.com/gh/owner/repo.git`) so don't trust `gh repo view` alone. Use this fallback chain:
   ```bash
   # Try gh first (works if remote is github.com)
   REPO=$(gh repo view --json nameWithOwner -q .nameWithOwner 2>/dev/null) || \
   # Fall back to parsing the remote URL for the github.com/owner/repo pattern
   REPO=$(git remote get-url origin | sed -E 's#.*github\.com/([^/]+/[^/]+)(\.git)?$#\1#') || \
   # Or look for any /owner/repo.git suffix and ask the user to confirm
   REPO=$(git remote get-url origin | sed -E 's#.*/([^/]+/[^/]+)\.git$#\1#')
   ```
   If still ambiguous, ask the user "this fork's GitHub identity is `<guess>`, correct?".

3. **Confirm it's a fork + capture metadata in one call**:
   ```bash
   gh api "repos/$REPO" --jq '{fork:.fork, parent:.parent.full_name, default:.default_branch}'
   ```
   Refuse to install if `.fork == false`.

4. **Probe the upstream for `.github/workflows/`** — this decides whether to push the user toward Step A immediately or defer it:
   ```bash
   gh api "repos/$UPSTREAM/contents/.github/workflows" --jq 'length' 2>/dev/null
   ```
   - Returns a number → upstream has workflows; the fallback will eventually fail. **Recommend Step A now.**
   - Returns 404 / error → no workflows folder; **safe to skip Step A** and rely on `GITHUB_TOKEN`. (Note: this is a snapshot; upstream could add workflows later, in which case the fallback handles the upgrade gracefully.)

5. **Read `.github/workflows/sync-fork.yml`** if it exists — don't silently overwrite a customised version. Diff against the template and ask before clobbering.

6. **Parse the user's schedule request.** "每 4 小时" → `*/4`; "每天两次" → `*/12`; "每 30 分钟" → `*/30 * * * *` (note: minute slot is busy, can't dodge stampede). Default: `17 */6 * * *`.

## How to install

### Step A — Provision SYNC_FORK_TOKEN (optional, recommended if upstream has workflows)

Skip this step if pre-flight #4 reported no `.github/workflows/` in upstream — the workflow will run on `GITHUB_TOKEN` alone and self-report when it eventually needs an upgrade.

**GitHub does NOT provide a public API to create PATs** (`POST /authorizations` was removed in 2020; fine-grained PATs require the web UI). Three options, in preference order:

**Option 1 (recommended, ~10s)** — reuse the user's gh CLI OAuth token:

```bash
# Refresh the CLI token to include workflow scope. Pops a browser
# tab once; user clicks "Authorize". Idempotent if already granted.
gh auth refresh -s workflow,repo

# Store the resulting token as the repo secret. gh auth token
# returns the OAuth token of the active gh CLI session.
gh secret set SYNC_FORK_TOKEN \
  -R <owner>/<repo> \
  --body "$(gh auth token)"
```

Tradeoff: the token is bound to the user's gh CLI session. If they `gh auth logout` or rotate creds, the workflow falls back to `GITHUB_TOKEN` (graceful — runs continue, just lose workflow-scope ability).

**Option 2 (production)** — fine-grained PAT via web UI:

Open `https://github.com/settings/personal-access-tokens/new`. Scope:
- Repository access: only `<owner>/<repo>`
- Permissions: Contents:RW + Workflows:RW

Then:
```bash
gh secret set SYNC_FORK_TOKEN -R <owner>/<repo>
# Paste token at the prompt, Enter, Ctrl-D
```

**Option 3 (org policy bans PATs)** — skip the secret entirely. The workflow runs on `GITHUB_TOKEN` and fails when upstream touches workflows; the user accepts that as a "I'll deal with it manually" trigger.

After provisioning, verify:
```bash
gh secret list -R <owner>/<repo> | grep SYNC_FORK_TOKEN
```

### Step B — Write the workflow file

1. Copy `templates/sync-fork.yml` from this skill's directory to `<repo>/.github/workflows/sync-fork.yml`. Use Read + Write tools (don't shell out — keeps it cross-platform).

2. **If default branch ≠ `main`**: edit the copied file's `BRANCH:` env var.

3. **If the cron schedule should differ** from `'17 */6 * * *'`: edit the cron line. Keep the minute non-zero (avoid the on-the-hour stampede). Common patterns:
   - Every 4 hours: `'17 */4 * * *'`
   - Twice daily: `'17 */12 * * *'`
   - Daily at ~3am UTC: `'17 3 * * *'`

The token fallback expression doesn't need to change regardless of whether Step A was done.

### Step C — Commit + push

**Check for project commit gates first.** Read `CLAUDE.md` / `AGENTS.md` / `CONTRIBUTING.md` at the repo root for pre-commit requirements (lint, format, tests, commitlint). If any exist:
- Mention them and ask whether to run or skip
- Don't silently bypass — the project owner put them there for a reason
- Workflow files don't usually trigger those checks but the gate may apply unconditionally

```bash
git add .github/workflows/sync-fork.yml
git commit -m "ci: add automatic fork sync from upstream every <N> hours"
git push
```

### Step D — Verify

```bash
# Trigger a manual run to confirm token + permissions work
gh workflow run sync-fork.yml -R <owner>/<repo>

# Watch the most recent run
RUN_ID=$(gh run list -R <owner>/<repo> --workflow sync-fork.yml --limit 1 --json databaseId -q '.[0].databaseId')
gh run watch "$RUN_ID" -R <owner>/<repo> --exit-status
```

**Reading the run log**: the first echoed line tells you which token was used:
- `Token: SYNC_FORK_TOKEN (custom PAT/App with workflow scope)` → robust path
- `Token: default GITHUB_TOKEN (no workflow scope)` → fallback path; will fail when upstream changes workflows

If it fails, the most likely causes (in order):
1. Upstream changed workflows + `SYNC_FORK_TOKEN` not set → run Step A, no file edit needed
2. `SYNC_FORK_TOKEN` set but missing scopes → re-run Step A with correct scopes
3. Settings → Actions → General → Workflow permissions is "Read-only" → toggle to "Read and write"
4. Branch protection rule blocks direct push → either grant a bypass or refactor to PR-based sync (separate skill)

## Substrate / write-surface gotchas

- **Myco substrates** (repos with `_canon.yaml` at the root): the `system.write_surface.allowed` list typically doesn't include `.github/**` by default. Before writing the workflow, check `_canon.yaml`. If `.github/**` is missing, add it to the `allowed` list — otherwise Myco's R6 write-surface guard rejects the file. After editing canon, run `myco_immune` to confirm no drift.

- **`.gitignore`-shadowed `.github/`**: rare but real — some org templates ignore the whole `.github/` dir. `git status` after `git add` should show the workflow file as new; if it doesn't, check `.gitignore`.

## What the workflow actually does at runtime

```
1. Selects token: SYNC_FORK_TOKEN if present, else GITHUB_TOKEN
2. Echoes which token was selected (so the run log says it)
3. gh api repos/$REPO --jq '.parent.full_name'  → prints upstream
4. gh repo sync $REPO --branch $BRANCH          → POST merge-upstream API
5. On failure with GITHUB_TOKEN, hints at provisioning SYNC_FORK_TOKEN
6. Exits 0 on fast-forward / already-up-to-date, non-zero on conflict
```

The `concurrency: sync-fork` group prevents two scheduled runs from racing. `cancel-in-progress: false` means a long-running merge isn't killed by the next tick.

## Known limitations

- **`GITHUB_TOKEN` pushes don't trigger downstream workflows.** Even after the sync succeeds, pushes via the default Actions token won't trigger other `on: push` workflows in the same repo (anti-recursion). PAT-based pushes (via SYNC_FORK_TOKEN) DO trigger them — so this is a second reason to provision the PAT if you have CI that should run after each sync.

- **Branch protection rules requiring PR-only updates** will block direct pushes from this workflow. Either grant the workflow a bypass, or refactor to "create a PR from upstream" instead of merging — different skill, different design.

- **Multiple branches**: this skill installs single-branch sync. To sync multiple branches, duplicate the `sync` job per branch in the same workflow file (cheap, ~2 lines per branch) — or convert `BRANCH` to a matrix. Don't promise this in a single shot — ask the user which branches.

## Output to user (after install)

Tell the user concisely:
1. File written: `.github/workflows/sync-fork.yml`
2. Schedule: `<the cron you used>` (translate to human English: "every 4 hours at minute 17")
3. Token strategy: which path you took
   - "PAT provisioned via Option 1 / 2 — workflow-scope merges enabled"
   - "No PAT — running on GITHUB_TOKEN. Will auto-upgrade when you set SYNC_FORK_TOKEN later (no file edit needed). Upstream's `.github/workflows/` is currently {empty/non-empty}."
4. Manual test: `https://github.com/<owner>/<repo>/actions/workflows/sync-fork.yml` (or the result of Step D if you already triggered it)
5. If you modified `_canon.yaml` or `.gitignore`, mention it.
6. If the user hasn't pushed yet, prompt them: workflows only register once they reach the default branch on origin.
