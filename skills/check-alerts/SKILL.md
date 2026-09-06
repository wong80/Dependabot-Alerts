---
name: check-alerts
description: Fetch, display, and fix Dependabot security alerts for a GitHub repo or org
argument-hint: "<owner/repo or org-name>"
allowed-tools:
  - PowerShell
  - Bash
  - Read
  - Grep
  - Glob
  - Edit
  - Write
  - AskUserQuestion
  - Agent
---

# Check Dependabot Alerts

Fetch Dependabot security alerts, display them in a prioritized table, and fix selected alerts by creating PRs.

## Step 1 — Prerequisite check

Run these checks via **PowerShell** (not Bash — `gh` is on the PowerShell PATH):

1. **`gh` CLI installed?**
   ```powershell
   gh --version
   ```
   If missing, tell the user to install it:
   - Windows: `winget install GitHub.cli` or `scoop install gh`
   - macOS: `brew install gh`
   - Linux: see https://github.com/cli/cli/blob/trunk/docs/install_linux.md

2. **Authenticated?**
   ```powershell
   gh auth status
   ```
   If not authenticated, tell the user to run `gh auth login`.

3. **`security_events` scope?**
   Check the output of `gh auth status` for the scopes listed. If `security_events` is not present, tell the user:
   > Run `gh auth refresh --scopes security_events` to add the required scope for reading Dependabot alerts.

   Stop here until prerequisites are met.

## Step 2 — Parse target

Read `$ARGUMENTS` to determine what to scan:

- **`owner/repo`** (contains `/`): scan that single repo.
- **`org-name`** (no `/`): fetch all repos in the org via:
  ```powershell
  gh api "/orgs/{org}/repos" --paginate --jq '.[].full_name'
  ```
- **Empty**: detect from the current directory's git remote:
  ```powershell
  git remote get-url origin
  ```
  Parse `owner/repo` from the URL. If not in a git repo, ask the user to provide a target.

## Step 3 — Fetch alerts

For each repo, fetch open Dependabot alerts:

```powershell
gh api "/repos/{owner}/{repo}/dependabot/alerts?state=open&per_page=100" --paginate
```

Extract these fields from each alert:
- `number`
- `security_vulnerability.severity` → **severity**
- `dependency.package.name` → **package**
- `dependency.package.ecosystem` → **ecosystem**
- `dependency.scope` → **scope** (runtime / development / null)
- `dependency.relationship` → **relationship** (direct / transitive)
- `dependency.manifest_path` → **manifest**
- `security_vulnerability.first_patched_version.identifier` → **patched_version** (may be null)
- `security_advisory.ghsa_id` → **ghsa_id**
- `security_advisory.summary` → **summary**

If a repo returns zero alerts, skip it silently (don't display an empty table).

## Step 4 — Detect existing Dependabot PRs

For each repo that has alerts, check for existing fix PRs:

```powershell
gh pr list --repo {owner}/{repo} --label dependencies --state open --json number,headRefName,title --jq '.[] | {number, headRefName, title}'
```

Match each PR's `headRefName` against the pattern `dependabot/{ecosystem}/{package}`. If a match is found, tag the corresponding alert with `PR #{number} exists`.

## Step 5 — Display

Build a markdown table sorted by severity (critical → high → medium → low).

**Columns:** `#` | `Severity` | `Package` | `Ecosystem` | `Scope` | `Relationship` | `Patched Version` | `Notes`

- **Scope**: show `runtime`, `dev`, or `—` for null
- **Relationship**: show `direct`, `transitive`, or `—`
- **Patched Version**: show the version, or `no fix available` if null
- **Notes**: show `PR #N exists` if detected, otherwise empty

**For org scans**: group alerts under repo headers with per-repo counts:
```
### owner/repo-name (3 alerts)
| # | Severity | Package | ... |
```

**Summary** at the bottom:
```
Total: X alerts (C critical, H high, M medium, L low)
```

## Step 6 — Selection

Use `AskUserQuestion` to let the user pick which alerts to fix. Offer options like:

- Fix specific alerts by number (e.g., "1, 5, 12")
- "All critical"
- "All critical and high"
- "All critical runtime" (critical severity + runtime scope)
- "All direct" (all direct dependencies)
- "All"
- "None" (just wanted to see the report)

**Exclude** alerts where `patched_version` is null — these cannot be auto-fixed.

If the user selects alerts tagged with "PR exists", proceed anyway (the user chose to override).

## Step 7 — Fix loop

For each selected alert, invoke the **dependabot-fixer** skill with these parameters:
- `owner/repo`
- `alert_number`
- `package_name`
- `ecosystem`
- `manifest_path`
- `patched_version`
- `dependency_relationship` (direct or transitive)
- `current_version` (from the vulnerable range if parseable)

Track results as you go:
- **Success**: record the PR URL
- **Failure**: record the error, continue to the next alert

## Step 8 — Summary report

After all fixes are attempted, display a summary:

```
## Fix Summary

### Succeeded
- ✅ lodash 4.17.19 → 4.17.21 — PR #123 (https://github.com/...)
- ✅ express 4.17.0 → 4.18.2 — PR #124

### Failed
- ❌ webpack 4.44.0 — push rejected: branch protection requires review
- ❌ minimist 1.2.0 — transitive dependency, no update command available

### Skipped
- ⏭️ node-fetch — no patched version available
```
