# Plan: `dependabot-alerts` Claude Code Plugin

## Context

You have GitHub repos with Dependabot security alerts that need fixing. Today, you'd manually go to each repo on GitHub, review each alert, open the affected manifest file, bump the version, commit, push, and create a PR — tedious and error-prone across multiple repos.

This plugin automates the entire workflow in two modes:
1. **Interactive** (`/check-alerts`): fetch alerts, display them, let you pick which to fix, apply the update locally, and create a PR.
2. **Automated** (GitHub Actions reusable workflow): auto-fix qualifying alerts the moment they appear — no human trigger needed.

## Approach

Build a Claude Code plugin with two skills (one user-invoked slash command, one model-invoked fixer) plus a reusable GitHub Actions workflow for automated fixes. The SKILL.md files instruct Claude step-by-step — Claude runs `gh`/`git` commands directly via PowerShell/Bash rather than through wrapper scripts, keeping things simple and avoiding PATH issues (`gh` is in PowerShell, not Git Bash).

## Plugin File Tree

```
Dependabot-Alerts/
├── .claude-plugin/
│   └── plugin.json                                # Plugin manifest
├── .github/
│   └── workflows/
│       └── auto-fix-alert.yml                     # Reusable workflow: auto-fix on alert
├── skills/
│   ├── check-alerts/
│   │   ├── SKILL.md                               # /check-alerts slash command
│   │   └── references/
│   │       └── gh-api-reference.md                # Dependabot API endpoints + response schema
│   └── dependabot-fixer/
│       ├── SKILL.md                               # Model-invoked: applies a single fix
│       └── references/
│           └── package-managers.md                # Update commands per ecosystem
└── docs/
    └── agents/                                    # Agent skill config (already created)
```

**Total: 6 files to create.** No shell scripts, no subagents.

## Design Decisions

### Interactive plugin (`/check-alerts`)

| # | Decision | Answer |
|---|----------|--------|
| Q1 | Target scope | Three modes: `owner/repo`, org name, or auto-detect from current dir |
| Q2 | Existing Dependabot PRs | Detect via `gh pr list` per repo, warn in table, still selectable |
| Q3 | Version selection | Minimum patched version from `first_patched_version.identifier` |
| Q4 | PR granularity | One PR per alert (atomic) |
| Q5 | Local repo discovery | Hardcoded `C:\Users\zhiyu\Documents\GitHub\{repo-name}` |
| Q6 | Lock file regeneration | Best-effort — run install command if runtime available, warn if not |
| Q7 | Transitive deps | Detect via `dependency.relationship`, run ecosystem update or skip with warning |
| Q8 | PR detection method | One `gh pr list --label dependencies` per repo, match branch names |
| Q9 | Auth scope | Guide user to run `gh auth refresh --scopes security_events` if missing |
| Q10 | Alert dismissal | Leave open — GitHub auto-resolves when fix merges |
| Q11 | Scope column | Show runtime/dev scope in table, support filtering (e.g. "all critical runtime") |
| Q12 | Ecosystem coverage | All 9 ecosystems in reference from day one |
| Q13 | Error recovery | Continue on failure, report all failures at end |
| Q14 | Branch collision | Append `-fix` suffix if `dependabot/{ecosystem}/{package}` exists on remote |
| Q15 | No fix available | Show in table marked "no fix available", exclude from selection |
| Q16 | Org display | Group alerts by repo with headers and per-repo counts |
| Q17 | Subagent | Dropped — check-alerts handles full flow including batch fixes |
| Q18 | Worktree cleanup | Auto-remove on failure (no debris) |

### Auto-fix workflow

| # | Decision | Answer |
|---|----------|--------|
| Q19 | Trigger | `schedule` (cron every 6h) + `workflow_dispatch` — `dependabot_alert` is webhook-only, not a valid Actions trigger |
| Q20 | Scope | Critical/high severity, direct deps only, known patched version required |
| Q21 | Approval | Fully autonomous PR creation (PR is the review gate) |
| Q22 | Location | Reusable workflow in this repo, called via `uses:` |
| Q23 | Auth | `GITHUB_TOKEN` with `contents: write`, `pull-requests: write` (Dependabot alerts readable via `github-script` with default token) |
| Q24 | Runtime in CI | Install dynamically per ecosystem (`actions/setup-node`, etc.) |
| Q25 | Inputs | Severity filter (default: critical,high) and extra PR labels |
| Q26 | Auto-merge | No — PR is the human review point |
| Q27 | Failure notification | Actions tab only (no issue creation) |
| Q28 | Concurrency | Serialized via `concurrency: dependabot-auto-fix` group |

### API facts

- Auth requires `security_events` scope — not included in default `gh auth login`
- `dependency.relationship` field: `direct` | `transitive` | `unknown` | `inconclusive` | `null`
- `dependency.scope` field: `runtime` | `development` | `null`
- No "existing PR" field in alert API — detection requires separate `gh pr list`
- Alerts can be dismissed via `PATCH` with reason, but we leave them open (auto-resolve on merge)
- Neither `dependabot_alert` nor `repository_vulnerability_alert` is a valid GitHub Actions trigger — they are webhook-only events. Auto-fix uses `schedule` + `workflow_dispatch` instead

## Files to Create

### 1. `.claude-plugin/plugin.json`
Minimal manifest with name, version, description, author, keywords.

### 2. `skills/check-alerts/SKILL.md` (user-invoked)
Frontmatter: `name: check-alerts`, `argument-hint: <owner/repo or org>`, `allowed-tools` including PowerShell, Bash, Read, Grep, Glob, AskUserQuestion.

Body instructs Claude to:
1. **Prerequisite check** — run `gh --version` and `gh auth status` via PowerShell. If missing, guide installation. If not authed, guide `gh auth login`. Check for `security_events` scope — if missing, tell user to run `gh auth refresh --scopes security_events`.
2. **Parse arguments** — if `$ARGUMENTS` is `owner/repo`, use it directly. If it's an org name (no `/`), fetch all repos via `gh api /orgs/{org}/repos`. If empty, detect from current git remote.
3. **Fetch alerts** — `gh api /repos/{owner}/{repo}/dependabot/alerts --paginate` with jq filter to extract: number, severity, package name, ecosystem, scope (runtime/dev), relationship (direct/transitive), vulnerable range, patched version, GHSA ID, summary, manifest_path.
4. **Detect existing PRs** — `gh pr list --label dependencies --json headRefName,number,state` per repo. Match branch names against `dependabot/{ecosystem}/{package}` pattern. Tag matching alerts.
5. **Display** — markdown table sorted by severity (critical first). Columns: #, severity, package, ecosystem, scope, patched version, relationship, notes (existing PR / no fix). If org scan, group by repo with headers. Show summary counts. Alerts with `first_patched_version: null` marked "no fix available".
6. **Selection** — use AskUserQuestion with options: by number, "all critical", "all critical runtime", "all", etc. Exclude "no fix available" alerts.
7. **Fix loop** — for each selected alert, invoke the dependabot-fixer skill. On error, log failure, continue to next. After all, show summary report with PR URLs and any failures.

### 3. `skills/check-alerts/references/gh-api-reference.md`
Documents:
- `GET /repos/{owner}/{repo}/dependabot/alerts` — params: state, severity, ecosystem, sort, direction, per_page
- `GET /orgs/{org}/dependabot/alerts` — org-wide
- Response schema: key fields including `dependency.relationship`, `dependency.scope`, `security_vulnerability.first_patched_version.identifier`
- Required token scopes: `security_events` (not just `repo`)
- `PATCH /repos/{owner}/{repo}/dependabot/alerts/{alert_number}` for dismissal (documented but not used)

### 4. `skills/dependabot-fixer/SKILL.md` (model-invoked)
Frontmatter: `name: dependabot-fixer`, description with trigger phrases.

Body instructs Claude through the fix workflow:

**Step 1 — Locate repo locally:**
- Search `C:\Users\zhiyu\Documents\GitHub\{repo-name}` and current directory
- Verify via `git remote get-url origin`
- If found → use it. If not → `gh repo clone` into the GitHub directory

**Step 1b — Worktree (if needed):**
- For multiple alerts on same repo: use `git worktree add` for each fix branch
- Worktree path: `../{repo}-dependabot-{package}`

**Step 2 — Create fix branch:**
- `git fetch origin`, detect default branch
- `git checkout -b dependabot/{ecosystem}/{package-name}` from default branch
- If branch exists on remote → use `dependabot/{ecosystem}/{package-name}-fix` instead

**Step 3 — Apply dependency update:**
- Check `dependency.relationship`:
  - **Direct**: read manifest at `dependency.manifest_path`, bump to minimum patched version, run install/lock command if runtime available, warn if not
  - **Transitive**: run ecosystem's sub-dependency update command (e.g., `npm update {pkg}`), or skip with warning
- Consult `references/package-managers.md` for ecosystem-specific commands

**Step 4 — Commit, push, PR:**
- `git add` changed manifest + lockfile
- Commit message: `Bump {package} from {old} to {new}`
- `git push -u origin {branch-name}`
- `gh pr create --title "Bump {package} from {old} to {new}" --body "..." --label dependencies`
- Report the PR URL

**Step 5 — Cleanup:**
- If worktree was used: `git worktree remove` (both on success and failure)

### 5. `skills/dependabot-fixer/references/package-managers.md`
For each ecosystem, documents manifest file, lockfile, update command, version format, and how to find/edit the version string. Covers all 9 ecosystems: npm, pip, maven, gomod, cargo, composer, nuget, rubygems, github-actions.

### 6. `.github/workflows/auto-fix-alert.yml` (reusable workflow)
Triggered by `dependabot_alert: [created]`. Called by other repos via `uses: wong80/Dependabot-Alerts/.github/workflows/auto-fix-alert.yml@main`.

**Inputs:**
- `severity-filter` (default: `critical,high`)
- `extra-labels` (default: empty)

**Permissions:**
```yaml
permissions:
  security_events: read
  contents: write
  pull-requests: write
```

**Concurrency:** `concurrency: dependabot-auto-fix` (serialized)

**Logic:**
1. Read the alert from the event payload
2. Filter: skip if severity not in `severity-filter`, skip if `dependency.relationship` != `direct`, skip if `first_patched_version` is null
3. Detect default branch, check for branch name collision
4. Install ecosystem runtime dynamically (`actions/setup-node`, `actions/setup-python`, `actions/setup-go`, etc.)
5. Edit manifest to bump to minimum patched version
6. Regenerate lockfile
7. Commit, push, create PR with `GITHUB_TOKEN`

## Workflow Summary

### Interactive
```
/check-alerts owner/repo
  → prerequisite check (gh CLI, auth, security_events scope)
  → fetch alerts via gh api
  → detect existing Dependabot PRs via gh pr list
  → display markdown table (grouped by repo for org scans)
  → user picks which to fix (with scope/severity filtering)
  → for each selected alert:
      → find repo locally OR clone
      → create branch (with -fix suffix on collision)
      → update dependency (direct: manifest edit, transitive: ecosystem update)
      → regenerate lockfile if runtime available
      → commit, push, create PR
      → cleanup worktree if used (even on failure)
  → show summary with PR URLs + failures
```

### Automated
```
schedule (every 6h) or workflow_dispatch
  → fetch all open alerts via API
  → filter: critical/high + direct + has patched version
  → deduplicate by package
  → skip if fix branch/PR already exists
  → for each eligible package:
      → create branch, bump manifest
      → commit, push, create PR
  → summary of succeeded/failed/skipped
```

## Verification

1. **Plugin structure** — install locally, verify `/check-alerts` appears in `/help`
2. **Prerequisites** — run without `gh` CLI installed, verify it guides installation; run without `security_events` scope, verify it guides `gh auth refresh`
3. **Fetch + display** — run against a repo with known alerts, verify table columns (severity, scope, relationship, existing PR detection)
4. **Fix workflow** — pick one direct alert, verify branch creation, manifest edit, lockfile regen, push, PR creation
5. **Transitive dep** — pick a transitive alert, verify it runs ecosystem update command or skips with warning
6. **Branch collision** — test against a repo where `dependabot/{ecosystem}/{pkg}` branch exists, verify `-fix` suffix
7. **Clone scenario** — test against a repo not cloned locally, verify it clones and handles worktree
8. **Multi-alert** — test fixing multiple alerts, verify continue-on-failure and summary report
9. **Org scan** — test with an org name, verify grouped display and cross-repo fixing
10. **Auto-fix workflow** — push workflow to a test repo, trigger a Dependabot alert, verify autonomous PR creation
