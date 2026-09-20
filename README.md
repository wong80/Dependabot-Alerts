# dependabot-alerts

A Claude Code plugin that fetches, triages, and fixes Dependabot security alerts across your GitHub repos — interactively or automatically.

## What it does

- **`/check-alerts`** — Scan a repo, org, or your current directory for open Dependabot alerts. Displays a prioritized table with severity, scope (runtime/dev), relationship (direct/transitive), and existing PR detection. Pick which alerts to fix and it creates branches, bumps dependencies, and opens PRs.
- **Auto-fix workflow** — A reusable GitHub Actions workflow that runs on a schedule (every 6 hours) or on demand, automatically creating fix PRs for critical/high direct dependencies with a known patched version.

## Prerequisites

Before installing, make sure you have:

1. **Claude Code** — [Claude Code CLI](https://docs.anthropic.com/en/docs/claude-code) or the desktop app
2. **GitHub CLI (`gh`)** — [Install instructions](https://cli.github.com/)
   - Windows: `winget install GitHub.cli` or `scoop install gh`
   - macOS: `brew install gh`
   - Linux: see the [official docs](https://github.com/cli/cli/blob/trunk/docs/install_linux.md)
3. **`gh` authenticated** with the `security_events` scope:
   ```bash
   gh auth login
   gh auth refresh --scopes security_events --hostname github.com
   ```

## Installation

### Option 1: Plugin marketplace (recommended)

Run these two commands in Claude Code:

```
/plugin marketplace add https://github.com/wong80/Dependabot-Alerts.git
```

```
/plugin install dependabot-alerts@wong80
```

### Option 2: Install from local clone

Clone the repo, then point Claude Code at it:

```bash
git clone https://github.com/wong80/Dependabot-Alerts.git
```

```
/plugin install /path/to/Dependabot-Alerts
```

### Verify installation

After installing, run `/help` in Claude Code — you should see `check-alerts` listed as an available skill. You can also try:

```
/check-alerts
```

If you're in a git repo with Dependabot enabled, it will auto-detect the repo and fetch alerts.

## Usage

### Interactive: `/check-alerts`

```
/check-alerts                    # scan current repo (auto-detected from git remote)
/check-alerts owner/repo         # scan a specific repo
/check-alerts my-org             # scan all repos in an org
```

The skill walks you through:

1. **Prerequisite check** — verifies `gh` CLI, authentication, and token scopes
2. **Fetch alerts** — pulls all open Dependabot alerts via the GitHub API
3. **Detect existing PRs** — flags alerts that already have a Dependabot PR open
4. **Display** — markdown table sorted by severity, with columns for scope, relationship, and patched version
5. **Selection** — pick alerts to fix by number, severity, scope, or "all"
6. **Fix** — for each selected alert: clones the repo if needed, creates a branch, bumps the dependency in the manifest, regenerates the lockfile (if the runtime is available), commits, pushes, and opens a PR
7. **Summary** — lists all created PRs and any failures

### Automated: GitHub Actions workflow

To use the auto-fix workflow in your own repo, create this file:

```yaml
# .github/workflows/dependabot-auto-fix.yml
name: Dependabot Auto-Fix

on:
  schedule:
    - cron: "0 */6 * * *"    # every 6 hours
  workflow_dispatch:           # manual trigger

jobs:
  fix:
    uses: wong80/Dependabot-Alerts/.github/workflows/auto-fix-alert.yml@main
```

**What it does:**
- Runs every 6 hours (or manually via "Run workflow" in the Actions tab)
- Fetches all open Dependabot alerts
- Filters to critical/high severity, direct dependencies only, with a known patched version
- Deduplicates by package (one PR per package)
- Skips packages that already have a fix branch or PR open
- Creates a PR automatically — the PR is the review gate (no auto-merge)

**Required permissions:** The calling repo's `GITHUB_TOKEN` must have `contents: write` and `pull-requests: write`. If your repo uses the default token permissions, you may need to update them under **Settings → Actions → General → Workflow permissions → Read and write permissions**.

**Customization** via workflow inputs:

| Input | Default | Description |
|-------|---------|-------------|
| `severity-filter` | `critical,high` | Comma-separated severities to auto-fix |
| `extra-labels` | _(empty)_ | Extra labels to add to the created PR |

Example with custom inputs:

```yaml
jobs:
  fix:
    uses: wong80/Dependabot-Alerts/.github/workflows/auto-fix-alert.yml@main
    with:
      severity-filter: "critical,high,medium"
      extra-labels: "automated,security"
```

## Supported ecosystems

| Ecosystem | Manifest | Lockfile regen |
|-----------|----------|----------------|
| npm | `package.json` | `npm install` |
| pip | `requirements.txt` / `pyproject.toml` | — |
| Maven | `pom.xml` | — |
| Go modules | `go.mod` | `go mod tidy` |
| Cargo | `Cargo.toml` | `cargo update` |
| Composer | `composer.json` | `composer install` |
| NuGet | `*.csproj` | `dotnet restore` |
| RubyGems | `Gemfile` | `bundle install` |
| GitHub Actions | `.github/workflows/*.yml` | — |

If the ecosystem runtime isn't installed locally, the plugin updates the manifest and warns that the lockfile wasn't regenerated.

## How it handles edge cases

- **Transitive dependencies** — detected via the `dependency.relationship` API field. The plugin runs the ecosystem's targeted update command (e.g., `npm update <pkg>`) or skips with a warning.
- **No patched version** — alerts with `first_patched_version: null` are shown in the table but excluded from fix selection.
- **Branch collision** — if a `dependabot/{ecosystem}/{package}` branch already exists, the plugin appends `-fix` to the branch name.
- **Multiple alerts, same package** — bumping to the highest patched version resolves all alerts at once.
- **Batch failures** — if one fix fails, the plugin continues to the next alert and reports all failures at the end.
- **Worktree isolation** — when fixing multiple alerts on the same repo, worktrees keep each fix branch isolated and are cleaned up automatically.

## Troubleshooting

### `gh: command not found`

The GitHub CLI isn't installed or isn't on your PATH.

**Fix:** Install `gh` using the instructions in [Prerequisites](#prerequisites). After installing, restart your terminal and verify with `gh --version`.

### `HTTP 403: Resource not accessible by personal access token`

Your `gh` token doesn't have the `security_events` scope, which is required to read Dependabot alerts.

**Fix:**
```bash
gh auth refresh --scopes security_events --hostname github.com
```

Then retry `/check-alerts`. You can verify your scopes with `gh auth status`.

### `HTTP 403` when using the auto-fix workflow

The workflow's `GITHUB_TOKEN` doesn't have write permissions.

**Fix:** Go to your repo's **Settings → Actions → General → Workflow permissions** and select **Read and write permissions**. The workflow needs `contents: write` and `pull-requests: write`.

### `/check-alerts` doesn't appear in `/help`

The plugin isn't installed properly.

**Fix:**
1. Make sure you ran both commands: `marketplace add` and then `plugin install`
2. Restart Claude Code after installing
3. Run `/plugin list` to confirm `dependabot-alerts` appears
4. If installing from a local path, make sure the path points to the directory containing `.claude-plugin/plugin.json`

### `No alerts found` but I know there are alerts

A few things to check:

1. **Dependabot is enabled** on the repo — go to **Settings → Code security → Dependabot alerts** and make sure it's turned on
2. **Alerts are open** — the plugin only fetches alerts with `state=open`. Dismissed or fixed alerts won't appear
3. **Correct repo** — if you ran `/check-alerts` without arguments, it auto-detects from `git remote`. Make sure you're in the right directory, or pass `owner/repo` explicitly

### Push rejected / branch protection errors

If the target repo has branch protection rules, the plugin's push or PR creation may fail.

**Fix:** The plugin creates feature branches (not pushing to `main` directly), so pushes should succeed. But if branch protection blocks the push entirely:
- Check that your `gh` token has `repo` scope (not just `security_events`)
- If your org requires signed commits, configure git signing locally
- If there's a branch restriction on who can push, you may need to create the PR manually

### Lockfile not regenerated (CI fails)

The plugin warns `⚠️ {ecosystem} runtime not available — manifest updated but lockfile not regenerated` when the language runtime (Node.js, Go, Python, etc.) isn't installed on your machine.

**Fix:**
- Install the required runtime locally and re-run the fix, or
- After the PR is created, run the install command in the PR branch manually:
  - npm: `npm install`
  - Go: `go mod tidy`
  - NuGet: `dotnet restore`
  - Cargo: `cargo update`
  - Composer: `composer install`
  - RubyGems: `bundle install`

### Transitive dependency can't be auto-fixed

Transitive dependencies aren't declared in your manifest — they're pulled in by another package. The plugin tries the ecosystem's targeted update command (e.g., `npm update <pkg>`), but this may not always work.

**Fix:**
- Identify which direct dependency pulls in the vulnerable transitive package (check `npm ls <pkg>` or equivalent)
- Update that direct dependency to a version that uses a patched transitive
- If no update path exists, consider adding a `resolution` or `override` in your manifest

### Auto-fix workflow runs but creates no PRs

Check the workflow run logs in your repo's **Actions** tab. Common reasons:

1. **No eligible alerts** — only critical/high severity, direct dependencies with a known patched version qualify (by default)
2. **PR already exists** — the workflow skips packages that already have an open PR or fix branch
3. **Unsupported ecosystem** — the workflow currently handles npm, pip, Go, NuGet, and GitHub Actions in CI; other ecosystems are skipped

### Auto-fix workflow doesn't trigger

The workflow uses `schedule` (cron) and `workflow_dispatch` triggers.

- **Schedule:** GitHub Actions schedules can be delayed during high load. The `0 */6 * * *` cron runs at 00:00, 06:00, 12:00, 18:00 UTC. GitHub may also disable scheduled workflows on repos with no recent activity (after 60 days of inactivity).
- **Manual:** Go to **Actions → Dependabot Auto-Fix → Run workflow** to trigger it on demand.

### Multiple alerts for the same package

This is normal — one vulnerable package version can have multiple CVEs. The plugin (and the auto-fix workflow) deduplicates by package, bumping to the highest required patched version. One PR resolves all alerts for that package.

## Project structure

```
Dependabot-Alerts/
├── .claude-plugin/
│   ├── plugin.json              # Plugin manifest
│   └── marketplace.json         # Marketplace metadata
├── .github/workflows/
│   └── auto-fix-alert.yml       # Reusable auto-fix workflow
├── skills/
│   ├── check-alerts/
│   │   ├── SKILL.md             # /check-alerts slash command
│   │   └── references/
│   │       └── gh-api-reference.md
│   └── dependabot-fixer/
│       ├── SKILL.md             # Model-invoked single-alert fixer
│       └── references/
│           └── package-managers.md
├── docs/
│   └── agents/                  # Agent skill config
│       ├── issue-tracker.md
│       ├── triage-labels.md
│       └── domain.md
├── CLAUDE.md                    # Project instructions
├── CONTEXT.md                   # Domain glossary
├── PLAN.md                      # Design decisions
└── README.md
```

## License

MIT
