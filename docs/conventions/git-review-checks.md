# Git/GitHub Review Checks

Standalone checklist for auditing a project's git, gh CLI, and GitHub.com usage.
Consumable by any repo via `dc:sync-conventions`. Maintained by git-organizer (authority).

Source: `.claude/skills/git-review/references/check-categories.md`

## Scoring

Each category scores 0-100 based on checks passed:

| Score | Rating | Meaning |
|-------|--------|---------|
| 90-100 | Excellent | Meets or exceeds best practices |
| 70-89 | Good | Minor gaps, low risk |
| 50-69 | Fair | Notable gaps, should address |
| 0-49 | Needs Work | Significant issues, prioritize fixes |

Overall score is a weighted average:

| Category | Weight | Focus |
|----------|--------|-------|
| A. API Usage | 20% | gh CLI and API patterns |
| B. Actions Security | 25% | Workflow hardening |
| C. Authentication | 20% | Token and credential hygiene |
| D. Git Hygiene | 15% | Branch, commit, merge practices |
| E. Rate Limits | 10% | API budget and throttling |
| F. Error Recovery | 10% | Resilience patterns |

## A. API Usage

How well the project uses `gh` CLI and GitHub API.

- [ ] Every `gh api` call uses `--jq` for server-side filtering
- [ ] List endpoints use `--paginate`
- [ ] No raw `curl` with tokens (use `gh api` instead)
- [ ] API responses cached locally with freshness checks
- [ ] No hardcoded API versions unless necessary
- [ ] `--header` used for API version pinning when needed
- [ ] Read-only `gh api` calls use `--cache` to reduce rate limit consumption

## B. Actions Security

GitHub Actions workflow hardening.

- [ ] All third-party actions pinned by full SHA, not tag
- [ ] `permissions:` block present and uses least privilege
- [ ] No `pull_request_target` without careful review
- [ ] Concurrency groups prevent redundant runs
- [ ] Secrets not exposed via `echo` or `cat` in workflow steps
- [ ] `GITHUB_TOKEN` scope minimized per job
- [ ] Reusable workflows use `workflow_call` trigger correctly
- [ ] `paths-ignore` or event filters avoid unnecessary triggers

## C. Authentication

Token and credential management.

- [ ] Fine-grained PAT used (not classic token)
- [ ] Token permissions documented
- [ ] SSH remote configured for push operations
- [ ] `GH_TOKEN` sourced from environment, never hardcoded
- [ ] `gh auth status` reports expected scopes
- [ ] No secrets in git history (gitleaks or similar pre-commit hook)

## D. Git Hygiene

Core git practices (forge-independent).

- [ ] Default branch is `main`
- [ ] No force push to main/master in any script
- [ ] Squash merge configured (check repo settings via API)
- [ ] Auto-delete branches after merge enabled
- [ ] `.gitignore` covers common artifacts (.env, *.pem, node_modules, .tmp/, etc.)
- [ ] Conventional commit format in recent history
- [ ] Branch naming follows conventions (`feat/`, `fix/`, `chore/`, etc.)
- [ ] PR titles use conventional commit format

## E. Rate Limit Management

API budget and throttling patterns.

- [ ] Scripts have inter-request delays for bulk operations (>10 repos)
- [ ] Retry with exponential backoff for transient failures (429, 500, 502, 503)
- [ ] Rate limit remaining checked before bulk runs
- [ ] API call budget estimated for full-audit scenarios
- [ ] Secondary rate limits considered (search API, GraphQL complexity)

## F. Error Recovery

Resilience and error handling patterns.

- [ ] API errors categorized (rate limit vs auth vs not-found vs server error)
- [ ] Retry logic for transient failures
- [ ] Graceful degradation (skip item, continue run) on non-fatal errors
- [ ] Error output goes to stderr, not swallowed silently
- [ ] Exit codes meaningful (0=ok, 1=error, 2=dry-run)
- [ ] `set -euo pipefail` in all scripts

## How to Use

**Self-audit (any repo):**
Run checks against your own repo files, workflows, and `gh api` calls. Use `gh api` with `--cache` for API-based checks.

**Remote audit (git-organizer only):**
Use `gh api` to inspect another repo's workflows, settings, and recent commits. Note which checks require local filesystem access.

**Checklist mode:**
Run a single category (e.g., just Actions Security) for focused review.

## Output Format

```
## Git/GitHub Review: {project}
### Date: {YYYY-MM-DD}

### Summary
| Category | Score | Issues |
|----------|-------|--------|
| API Usage | {score}/100 | {count} |
| Actions Security | {score}/100 | {count} |
| Authentication | {score}/100 | {count} |
| Git Hygiene | {score}/100 | {count} |
| Rate Limits | {score}/100 | {count} |
| Error Recovery | {score}/100 | {count} |
| **Overall** | **{weighted}/100** | **{total}** |

### Findings (by severity)
#### Critical
{severity}: [{category}] {description}
  Location: {file:line or API endpoint}
  Fix: {specific remediation}

#### Warning
...

#### Info
...
```
