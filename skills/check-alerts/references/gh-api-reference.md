# Dependabot Alerts API Reference

## Endpoints

### List alerts for a repository

```
GET /repos/{owner}/{repo}/dependabot/alerts
```

**Parameters:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `state` | string | — | Filter by state: `auto_dismissed`, `dismissed`, `fixed`, `open` |
| `severity` | string | — | Filter by severity: `low`, `medium`, `high`, `critical` |
| `ecosystem` | string | — | Filter by ecosystem: `composer`, `go`, `maven`, `npm`, `nuget`, `pip`, `pub`, `rubygems`, `rust` |
| `scope` | string | — | Filter by scope: `development`, `runtime` |
| `sort` | string | `created` | Sort by: `created`, `updated` |
| `direction` | string | `desc` | Sort direction: `asc`, `desc` |
| `per_page` | integer | 30 | Results per page (max 100) |

**Example:**
```powershell
gh api "/repos/{owner}/{repo}/dependabot/alerts?state=open&per_page=100" --paginate
```

### List alerts for an organization

```
GET /orgs/{org}/dependabot/alerts
```

Same parameters as the repo endpoint, plus `repo` to filter by repo name. Returns alerts across all repos the token can access.

**Example:**
```powershell
gh api "/orgs/{org}/dependabot/alerts?state=open&severity=critical,high&per_page=100" --paginate
```

### Get a single alert

```
GET /repos/{owner}/{repo}/dependabot/alerts/{alert_number}
```

### Update an alert (dismiss/reopen)

```
PATCH /repos/{owner}/{repo}/dependabot/alerts/{alert_number}
```

**Body:**

| Field | Type | Description |
|-------|------|-------------|
| `state` | string | `dismissed` or `open` |
| `dismissed_reason` | string | Required when dismissing: `fix_started`, `inaccurate`, `no_bandwidth`, `not_used`, `tolerable_risk` |
| `dismissed_comment` | string | Optional, max 280 chars |

> **Note:** This plugin does not dismiss alerts. GitHub auto-resolves them when the vulnerable version is removed after a fix PR merges.

## Response Schema

### Alert object (key fields)

```json
{
  "number": 42,
  "state": "open",
  "dependency": {
    "package": {
      "ecosystem": "npm",
      "name": "lodash"
    },
    "manifest_path": "package.json",
    "scope": "runtime",
    "relationship": "direct"
  },
  "security_advisory": {
    "ghsa_id": "GHSA-xxxx-xxxx-xxxx",
    "cve_id": "CVE-2021-12345",
    "summary": "Prototype Pollution in lodash",
    "severity": "critical",
    "description": "..."
  },
  "security_vulnerability": {
    "severity": "critical",
    "vulnerable_version_range": ">= 4.0.0, < 4.17.21",
    "first_patched_version": {
      "identifier": "4.17.21"
    },
    "package": {
      "ecosystem": "npm",
      "name": "lodash"
    }
  },
  "url": "https://api.github.com/repos/owner/repo/dependabot/alerts/42",
  "html_url": "https://github.com/owner/repo/security/dependabot/42",
  "created_at": "2024-01-15T10:30:00Z",
  "dismissed_at": null,
  "fixed_at": null,
  "auto_dismissed_at": null
}
```

### Field details

**`dependency.scope`** — where the dependency is used:
- `"runtime"` — production dependency
- `"development"` — dev/test dependency
- `null` — scope could not be determined

**`dependency.relationship`** — how the dependency is included:
- `"direct"` — explicitly declared in the manifest
- `"transitive"` — pulled in as a sub-dependency
- `"unknown"` — relationship could not be determined
- `"inconclusive"` — analysis was inconclusive
- `null` — not available

**`security_vulnerability.first_patched_version`** — the minimum safe version:
- Object with `identifier` field (e.g., `"4.17.21"`) when a fix exists
- `null` when no patched version is known yet

## Required Auth Scopes

The default `gh auth login` scopes (`repo`, `read:org`, `gist`) do **not** include Dependabot alert access.

**Classic PATs / OAuth tokens:** require the `security_events` scope.

**Fine-grained PATs:** require the "Dependabot alerts" repository permission (read access minimum).

**To add the scope:**
```powershell
gh auth refresh --scopes security_events
```

## jq Filters

Extract a compact alert summary:
```powershell
gh api "/repos/{owner}/{repo}/dependabot/alerts?state=open&per_page=100" --paginate --jq '.[] | {number, severity: .security_vulnerability.severity, package: .dependency.package.name, ecosystem: .dependency.package.ecosystem, scope: .dependency.scope, relationship: .dependency.relationship, manifest: .dependency.manifest_path, patched: .security_vulnerability.first_patched_version.identifier, ghsa: .security_advisory.ghsa_id, summary: .security_advisory.summary}'
```
