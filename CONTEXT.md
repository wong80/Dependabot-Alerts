# Dependabot Alerts Plugin — Domain Glossary

## Terms

- **Alert**: A Dependabot security alert on a GitHub repository, representing a known vulnerability in a dependency. Identified by a numeric `number` scoped to the repo. Has a `state` (open, dismissed, fixed).

- **Severity**: One of `critical`, `high`, `medium`, `low`. Assigned by GitHub's security advisory database. Used for filtering and prioritization.

- **Ecosystem**: The package manager a dependency belongs to — `npm`, `pip`, `maven`, `gomod`, `cargo`, `composer`, `nuget`, `rubygems`, `github-actions`. Determines how to locate and edit the manifest.

- **Manifest**: The file declaring a dependency (`package.json`, `requirements.txt`, `go.mod`, etc.). The alert's `dependency.manifest_path` points to it.

- **Lockfile**: The file pinning exact resolved versions (`package-lock.json`, `go.sum`, etc.). Must be regenerated after a manifest edit for CI to pass. Not all ecosystems have one.

- **Direct dependency**: A package explicitly declared in the manifest. The plugin can bump its version directly.

- **Transitive dependency**: A package pulled in as a sub-dependency of a direct dependency. Not declared in the manifest — requires the ecosystem's update command (e.g., `npm update {pkg}`) rather than a manifest edit. Identified by `dependency.relationship: "transitive"` in the API response.

- **Patched version**: The minimum version of a package that resolves the vulnerability. Comes from `security_vulnerability.first_patched_version.identifier`. When `null`, no fix is available yet.

- **Scope**: Whether a dependency is used at runtime (`"runtime"`) or only during development (`"development"`). A critical vulnerability in a dev dependency is lower priority than in a runtime one.

- **Fix**: The act of bumping a dependency to its patched version, committing, pushing, and opening a PR. One fix per alert.

- **Reusable workflow**: A GitHub Actions workflow defined in this repo that other repos reference via `uses: wong80/Dependabot-Alerts/.github/workflows/auto-fix-alert.yml@main`. Fires on the `dependabot_alert` event to auto-fix qualifying alerts.

## Terms to avoid

- **"Vulnerability"** as a synonym for "alert" — an alert is the repo-specific notification; the vulnerability is the underlying CVE/GHSA. They are different things.
- **"Patch"** as a noun for the fix PR — use "fix PR" or "bump PR" to avoid confusion with patch-level semver bumps.
