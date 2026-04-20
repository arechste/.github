# Local CI Conventions

## Layer Model

CI validation happens in layers. Earlier layers are faster and cheaper. Later layers
are remote and consume GitHub Actions minutes.

```
Layer 0: Editor        — shellcheck LSP integration (instant, in-editor)
Layer 1: Pre-commit    — gitleaks, shellcheck, jq, trailing-ws (seconds, on commit)
Layer 2: Local CI      — act runs GH Actions workflows locally in Docker (seconds)
Layer 3: Remote CI     — GitHub Actions as safety net + PR-context checks (minutes)
Layer 4: Scheduled     — Audit automation, read-only weekly, write monthly
```

**Principle**: catch issues at the earliest layer possible. Remote CI (Layer 3) is a
safety net, not the primary validation mechanism.

## Prerequisites

| Tool | Install | License | Purpose |
|------|---------|---------|---------|
| shellcheck | `brew install shellcheck` | GPL-3.0 | Shell script linter |
| pre-commit | `brew install pre-commit` | MIT | Git hook framework |
| gitleaks | `brew install gitleaks` | MIT | Secret scanner |
| jq | `brew install jq` | MIT | JSON processor |
| act | `brew install act` | MIT | Local GitHub Actions runner |
| OrbStack | `brew install --cask orbstack` | Free (personal) | Docker runtime for act |

All tools are managed in the dotfiles Brewfile and deployed via `brew bundle`.

## Layer 1: Pre-commit Hooks

Hooks are defined in `.pre-commit-config.yaml` at the repo root.

```bash
# Install hooks (one-time per clone)
pre-commit install

# Run all hooks manually
pre-commit run --all-files
```

Hooks run automatically on `git commit`. They catch:
- Invalid JSON (`check-json`)
- Shell script issues (`shellcheck` at warning severity)
- Leaked secrets (`gitleaks`)
- Trailing whitespace and missing newlines

## Layer 2: Local CI with act

`act` reads `.github/workflows/` directly and runs them in Docker containers.

```bash
# Run all local-compatible checks
./tools/ci/local-check.sh

# Pre-commit only (no Docker needed)
./tools/ci/local-check.sh --pre-commit-only

# Act only (Docker must be running)
./tools/ci/local-check.sh --act-only

# Check GitHub Actions billing/usage
./tools/ci/local-check.sh --usage
```

### What runs locally vs remotely

| Workflow | Local (act) | Remote (Actions) | Reason |
|----------|-------------|------------------|--------|
| ShellCheck | Yes | Yes (safety net) | Pure linting, no context needed |
| Validate JSON | Yes | Yes (safety net) | Pure validation |
| Conventions | No | Yes | Needs PR title, labels, commit list from GitHub |
| Audit (read-only) | No | Weekly cron | Needs `gh` auth, hits API for 155 repos |
| Audit (write) | No | Monthly cron | Needs CROSS_REPO_PAT, modifies other repos |

### Configuration

Default act settings are in `.actrc`:
- Uses `catthehacker/ubuntu:act-latest` image (lighter than full GitHub runner)
- Auto-detects event type

### Limitations

- `act` does not support `workflow_call` (reusable workflows)
- No PR context available locally — convention-check cannot run
- Docker must be running (OrbStack or Docker Desktop)
- Some GitHub Actions may behave slightly differently in act's environment

## Layer 3: Remote CI

Remote workflows have concurrency groups to cancel redundant runs:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.head_ref || github.ref }}
  cancel-in-progress: true
```

Path filters ensure workflows only run when relevant files change.

### Workflow categories

Workflows are tagged with a comment at the top of each file:

- `# Category: project-ci` — code quality gates (shellcheck, validate-json, conventions)
- `# Category: automation` — audit and cross-repo operations

## Layer 4: Scheduled Automation

Split into two workflows to reduce minute consumption:

| Workflow | Schedule | Scope | Auth |
|----------|----------|-------|------|
| `auto-audit-readonly.yml` | Weekly (Mon 9am UTC) | Inventory, audits, scoring | `github.token` |
| `auto-audit-write.yml` | Monthly (1st Mon) + manual | Cross-repo syncs, issue filing | `CROSS_REPO_PAT` |

## GitHub Actions Usage Tracking

Free tier: 2000 minutes/month for private repos. Monitor with:

```bash
./tools/ci/local-check.sh --usage
```

Requires `user` scope (one-time setup):
```bash
gh auth refresh -h github.com -s user
```

Billing resets monthly on your GitHub account creation anniversary.
Manual check: https://github.com/settings/billing/summary

### Automated Weekly Check

A remote Claude Code trigger runs `budget-monitor.sh --quick` every Monday
09:00 UTC and files a `type/ci` issue in this repo when usage crosses 75%.

- Trigger: `weekly-ci-budget-check` (id `trig_01DQokuHKATzgjzasBPLcRwL`)
- Manage / update / disable: https://claude.ai/code/scheduled
- Thresholds: `<50%` silent, `50–74%` breakdown only, `75–89%` opens
  `priority/high` issue, `≥90%` opens `priority/critical` issue with
  disable recommendations. Core 4 repos (git-organizer, dotclaude,
  dotfiles, mac-organizer) are exempt from auto-disable recommendations.

Prefer updating the trigger (via the URL above or `RemoteTrigger` API)
over editing this section — the trigger is authoritative.

## Cross-Project Strategy

This section describes how the layer model is applied across all repos under arechste/ management. It exists to answer one question: **"When I add a check, where does it run?"**

### Decision heuristic

For every new check, walk this list top-to-bottom and stop at the first Yes:

1. **Can it run as a pure function of the working tree?** (no network, no PR context, no cross-repo state) → **Layer 1 (pre-commit)**. Shellcheck, JSON validity, trailing whitespace, gitleaks, markdown lint, JSON schema validation.
2. **Does it need tools installed in a specific OS image but no GitHub API?** → **Layer 2 (act)**. Language toolchains, compiled linters, integration tests against local services.
3. **Does it need PR metadata, labels, commit list, or cross-repo access?** → **Layer 3 (remote Actions)**. convention-check, label sync, cross-repo audits.
4. **Is it an ongoing maintenance task that doesn't map to a git event?** → **Layer 4 (scheduled)**. Inventory refresh, maturity scoring, security audits.

**If a check appears in multiple layers**, Layer 1 is the primary enforcer and the higher layer is the safety net. Never have Layer 3 be the only place a check runs if it *could* run at Layer 1.

### Per-category budget guidance

Repos are categorized in `data/repo-inventory.json`. Each category has a rough target for remote Actions minute consumption per month:

| Category | Target | Strategy |
|----------|--------|----------|
| infrastructure | ≤ 100 min/mo | PR-only triggers, aggressive path filters, weekly scheduled audits only |
| active | ≤ 50 min/mo | PR-only triggers, path filters, no cron |
| learning | ≤ 10 min/mo | Convention-check only, no workflows beyond the public mirror |
| dormant | 0 min/mo | Workflows disabled via `tools/maintenance/toggle-workflows.sh` |
| fork | 0 min/mo | Don't manage upstream-owned CI |
| profile | 0 min/mo | No workflows |

These are targets, not hard budgets. The hard budget is the 2000 min/month free-tier ceiling across all private repos combined — enforced by `tools/ci/budget-monitor.sh`.

### Anti-patterns to avoid

The following patterns are the usual cause of wasted minutes. `tools/audit/ci-workflow-audit.sh` flags them across all repos.

- **Double triggers**: `on: [push, pull_request]` without branch filters on `push` runs every workflow twice for PR branches (once on push, once on PR open). Fix: use `pull_request` alone, or restrict `push` to `branches: [main]`.
- **Missing concurrency**: every new commit on a PR starts fresh jobs instead of cancelling in-flight runs. Fix: add `concurrency.cancel-in-progress: true` keyed on `github.head_ref || github.ref`.
- **Missing path filters**: docs-only or data-only commits trigger full build/test runs. Fix: `paths-ignore: ['**.md', 'docs/**']` at minimum.
- **Hourly/daily cron schedules**: almost always wasteful for a personal workflow — weekly is usually enough. Fix: move to `cron: '0 9 * * 1'` (Mon 9 UTC) or `workflow_dispatch` with a manual trigger.
- **macOS / Windows runners**: 10× and 2× the minute cost of Linux. Fix: use `runs-on: ubuntu-latest` unless the test specifically requires the OS.
- **Unbounded matrix**: 4 OS × 3 language versions = 12× multiplier on every workflow run. Fix: shrink the matrix or gate it behind a `workflow_dispatch` tag.

### Auditing cross-project

Two tools work together:

- `./tools/audit/ci-workflow-audit.sh [--category <name>] [<repo>]` — static analysis of workflow YAML files. Flags the anti-patterns above. Output: `data/ci-workflow-audit.json`.
- `./tools/ci/budget-monitor.sh --breakdown` — live consumption data from the billing API. Sorts repos by minutes used.

Run the static audit first to find *potential* waste, then cross-reference with the budget monitor to see which potential waste is actually biting.

### Rollout order

When adopting these conventions on a new (or existing) repo:

1. Install pre-commit hooks locally (`pre-commit install`) — Layer 1 is free and runs everywhere.
2. Add `convention-check` via `docs/conventions/adopting-ci.md` — Layer 3 PR gate.
3. Run `./tools/audit/ci-workflow-audit.sh <repo>` — baseline the anti-patterns.
4. Fix flagged workflows one at a time.
5. Add the repo to the weekly `budget-monitor.sh` breakdown review.

## Future: Multi-Forge Support

When self-hosted forges (Gitea, GitLab) come into scope:

- **Gitea Actions**: uses `act_runner`, compatible with GitHub Actions syntax. Workflows
  written for GitHub Actions work with minimal changes on Gitea.
- **github-act-runner**: can register as a self-hosted runner for both GitHub and Gitea.
- **act** workflows are already portable to Gitea Actions.

This path is documented but deferred until multi-forge becomes an active requirement.
