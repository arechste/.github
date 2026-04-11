# Repo Standards

What a well-maintained repo looks like under git-organizer management.

## Maturity Score

Every non-fork repo is scored on a 0-100 scale:

| Dimension | Points | What's checked |
|-----------|--------|----------------|
| Canonical labels | 20 | All 30 canonical labels present |
| Convention-check CI | 20 | `.github/workflows/conventions.yml` exists |
| README | 15 | Exists with meaningful content (>500 bytes) |
| Community health | 15 | CODE_OF_CONDUCT, CONTRIBUTING, SECURITY present |
| Security | 15 | No high/critical findings in security audit |
| CODEOWNERS | 5 | File exists defining code ownership |
| Claude config | 10 | CLAUDE.md and/or .claude/settings.json present |

### Score Tiers

| Tier | Score | Meaning |
|------|-------|---------|
| High | 70-100 | Well-maintained, meets all major standards |
| Medium | 40-69 | Partially compliant, some gaps |
| Low | 0-39 | Needs attention, multiple gaps |

### Automated Actions

- All repos: maturity scores tracked in weekly audit
- Active + infrastructure repos scoring <50: auto-filed remediation issue
- Other categories: report only, human decides

## Repo Checklist

### Minimum (all repos)
- [ ] Canonical labels synced
- [ ] README exists with project description
- [ ] Covered by .github repo templates (public) or .github-private (private)

### Recommended (active repos)
- [ ] Convention-check CI workflow adopted
- [ ] CODEOWNERS file defining ownership
- [ ] CLAUDE.md with project-specific instructions
- [ ] Dependabot vulnerability alerts enabled
- [ ] Security audit clean (no high/critical findings)
- [ ] Squash merge only — enforce via `gh repo edit --enable-squash-merge --disable-merge-commit --disable-rebase-merge --squash-merge-commit-message pr-title`
- [ ] Auto-delete head branches after merge — `gh repo edit --delete-branch-on-merge`

### Ideal (infrastructure repos)
- [ ] All recommended items above
- [ ] .claude/settings.json with permission matrix
- [ ] `fetch.prune=true` in gitconfig — keeps local tracking refs clean automatically
- [ ] No open critical/high Dependabot alerts
- [ ] Pre-commit hooks configured
- [ ] Renovate installed for dependency updates (see [dependency-management.md](dependency-management.md))
- [ ] SSH remote configured (avoid credential issues)

### Reusable workflow providers (private repos only)

For private repos that host `on: workflow_call` workflows intended for cross-repo use:

- [ ] `actions/permissions/access` `access_level` is `"user"` (not `"none"`)
- [ ] Workflows published to public `arechste/.github` via `tools/maintenance/sync-github-templates.sh` (consumers should never reference the private path directly — see [adopting-ci.md § Hosting reusable workflows](adopting-ci.md#hosting-reusable-workflows))

## Repo Categories

| Category | Count | Default Autonomy | Expectations |
|----------|-------|-------------------|--------------|
| active | 10 | pr-only | Full checklist compliance |
| infrastructure | 13 | pr-only | Full checklist + CI |
| learning | 18 | assist | Minimum only |
| dormant | 49 | read-only | Minimum only |
| fork | 62 | read-only | No management (upstream-owned) |
| profile | 3 | read-only | Templates via .github repo |

## Commands

```bash
# Score a single repo
./tools/audit/repo-maturity.sh <repo-name>

# Score all repos
./tools/audit/repo-maturity.sh all

# Score and file issues for low-scoring active repos
./tools/audit/repo-maturity.sh --file-issues all
```
