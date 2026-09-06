# dependabot-alerts

A Claude Code plugin that fetches, triages, and fixes Dependabot security alerts across your GitHub repos — interactively or automatically.

## What it does

- **`/check-alerts`** — Scan a repo, org, or your current directory for open Dependabot alerts. Displays a prioritized table with severity, scope (runtime/dev), relationship (direct/transitive), and existing PR detection. Pick which alerts to fix and it creates branches, bumps dependencies, and opens PRs.
- **Auto-fix workflow** — A reusable GitHub Actions workflow that fires on new Dependabot alerts and automatically creates fix PRs for critical/high direct dependencies.

## Prerequisites

- [GitHub CLI (`gh`)](https://cli.github.com/) installed and authenticated
- The `security_events` scope added to your `gh` token:
  ```
  gh auth refresh --scopes security_events --hostname github.com
  ```

## Install

Add the marketplace, then install the plugin:

```
/plugin marketplace add https://github.com/wong80/Dependabot-Alerts.git
/plugin install dependabot-alerts@wong80
```

Verify it's installed — `/check-alerts` should appear in `/help`.

## Usage

### Interactive: `/check-alerts`

```
/check-alerts                    # scan current repo
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

Add this workflow to any repo to auto-fix qualifying alerts the moment they appear:

```yaml
# .github/workflows/dependabot-auto-fix.yml
name: Dependabot Auto-Fix
on:
  dependabot_alert:
    types: [created]
jobs:
  fix:
    uses: wong80/Dependabot-Alerts/.github/workflows/auto-fix-alert.yml@main
```

**What it does:**
- Triggers on new Dependabot alerts
- Filters to critical/high severity, direct dependencies, with a known patched version
- Installs the ecosystem runtime (Node.js, Python, Go, etc.)
- Bumps the dependency to the minimum safe version
- Creates a PR automatically

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

- **Transitive dependencies** — detected via the `dependency.relationship` API field. The plugin runs the ecosystem's targeted update command (e.g., `npm update <pkg>`) or skips with a warning if no automated fix is available.
- **No patched version** — alerts with `first_patched_version: null` are shown in the table but excluded from fix selection.
- **Branch collision** — if a `dependabot/{ecosystem}/{package}` branch already exists (e.g., from a stale Dependabot PR), the plugin appends `-fix` to the branch name.
- **Multiple alerts, same package** — bumping to the highest patched version resolves all alerts for that package at once.
- **Batch failures** — if one fix fails, the plugin continues to the next alert and reports all failures at the end.
- **Worktree isolation** — when fixing multiple alerts on the same repo, worktrees keep each fix branch isolated. Worktrees are cleaned up automatically.

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
├── CONTEXT.md                   # Domain glossary
└── PLAN.md                      # Design decisions
```

## License

MIT
