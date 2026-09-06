---
name: dependabot-fixer
description: "Fix a single Dependabot alert by updating the vulnerable dependency and creating a PR"
model-triggered-by:
  - "fix a Dependabot alert"
  - "update a vulnerable dependency"
  - "patch a security vulnerability"
  - "bump a dependency to fix a security alert"
allowed-tools:
  - PowerShell
  - Bash
  - Read
  - Edit
  - Write
  - Grep
  - Glob
---

# Dependabot Fixer

Apply a single Dependabot alert fix: update the vulnerable dependency in its manifest, regenerate the lockfile if possible, and create a PR.

This skill is invoked by `/check-alerts` for each selected alert. It receives these parameters:
- `owner/repo` — the GitHub repository
- `alert_number` — the Dependabot alert number
- `package_name` — the vulnerable package
- `ecosystem` — the package manager (npm, pip, maven, etc.)
- `manifest_path` — path to the manifest file declaring the dependency
- `patched_version` — the minimum safe version to bump to
- `dependency_relationship` — `direct` or `transitive`
- `current_version` — the currently installed vulnerable version (if known)

## Step 1 — Locate the repo locally

Search for the repo in this order:

1. **Current directory** — check if `git remote get-url origin` matches `owner/repo`.
2. **GitHub directory** — check `C:\Users\zhiyu\Documents\GitHub\{repo-name}`.
3. **Clone it** — if not found locally:
   ```powershell
   gh repo clone {owner}/{repo} "C:\Users\zhiyu\Documents\GitHub\{repo-name}"
   ```

Once located, `cd` into the repo directory for all subsequent commands.

### Worktree (multiple alerts on the same repo)

If the repo directory already has uncommitted changes or is on a non-default branch (indicating another fix is in progress), use a worktree:

```powershell
git worktree add "../{repo-name}-dependabot-{package_name}" -b dependabot/{ecosystem}/{package_name}
```

Track that a worktree was created so Step 5 can clean it up.

## Step 2 — Create fix branch

```powershell
git fetch origin
```

Detect the default branch:
```powershell
gh repo view {owner}/{repo} --json defaultBranchRef --jq '.defaultBranchRef.name'
```

Create and switch to the fix branch:
```powershell
git checkout -b dependabot/{ecosystem}/{package_name} origin/{default_branch}
```

**If the branch already exists on the remote** (e.g., from a stale Dependabot PR):
```powershell
git ls-remote --heads origin dependabot/{ecosystem}/{package_name}
```
If it returns output, use the suffixed name instead:
```powershell
git checkout -b dependabot/{ecosystem}/{package_name}-fix origin/{default_branch}
```

## Step 3 — Apply dependency update

### Direct dependencies (`dependency_relationship` = `direct`)

1. **Read the manifest** at `manifest_path`.
2. **Find the version string** for `package_name` in the manifest. Consult `references/package-managers.md` for the ecosystem-specific pattern.
3. **Edit the manifest** to replace the current version with `patched_version`.
4. **Regenerate the lockfile** if the ecosystem's runtime is available:

   Run the appropriate install/update command from `references/package-managers.md` via **PowerShell**. Example for npm:
   ```powershell
   npm install
   ```

   If the runtime is not installed (command not found), warn:
   > ⚠️ {ecosystem} runtime not available — manifest updated but lockfile not regenerated. CI may fail.

### Transitive dependencies (`dependency_relationship` = `transitive`)

The package is not directly declared in the manifest — it's a sub-dependency.

1. **Try the ecosystem's targeted update command** (via PowerShell):

   | Ecosystem | Command |
   |-----------|---------|
   | npm | `npm update {package_name}` |
   | pip | `pip install --upgrade {package_name}` (if in a venv) |
   | gomod | `go get {package_name}@v{patched_version}` |
   | cargo | `cargo update -p {package_name}` |
   | composer | `composer update {package_name}` |
   | rubygems | `bundle update {package_name}` |

2. **If the runtime is not installed**, or the update command fails, skip with a warning:
   > ⚠️ Cannot auto-fix transitive dependency {package_name}. Update the parent package that depends on it, or run `{command}` manually.

3. **Verify the fix** — after running the update, check if the lockfile now contains `patched_version` or higher. If not, warn that the fix may be incomplete.

## Step 4 — Commit, push, PR

Determine the branch name being used (original or `-fix` suffixed).

**Stage changes:**
```powershell
git add {manifest_path}
```
Also stage the lockfile if it was regenerated (check `git status` for modified lockfiles).

**Commit:**
```powershell
git commit -m "Bump {package_name} from {current_version} to {patched_version}"
```
If `current_version` is unknown, use:
```powershell
git commit -m "Bump {package_name} to {patched_version}"
```

**Push:**
```powershell
git push -u origin {branch_name}
```

**Create PR:**
```powershell
gh pr create --repo {owner}/{repo} --title "Bump {package_name} from {current_version} to {patched_version}" --body "Fixes Dependabot alert #{alert_number}.`n`nBumps [{package_name}](https://github.com/advisories/{ghsa_id}) to {patched_version} to resolve a {severity} severity vulnerability.`n`n---`n*Created by the dependabot-alerts plugin.*" --label dependencies
```

Report the PR URL back to the caller.

## Step 5 — Cleanup

**If a worktree was used:**
```powershell
git worktree remove "../{repo-name}-dependabot-{package_name}" --force
```
Do this regardless of whether the fix succeeded or failed.

**Return to the original directory** if you changed it.

**Report result:**
- On success: return `{status: "success", pr_url: "...", package: "...", version: "..."}`
- On failure: return `{status: "failed", package: "...", error: "..."}`
