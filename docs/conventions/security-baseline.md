# Security Baseline

Security standards for repos managed by git-organizer, designed for GitHub Free tier.

## Constraints

GitHub Free limits security features for private repos:
- No branch protection rules (requires Pro/Team)
- No Dependabot alerts API for private repos (403)
- No required reviewers or status checks enforcement
- Workaround: enforce conventions via CI checks + PR discipline

## Audit Checks

The security audit (`security-audit.sh`) checks:

### All Repos
| Check | Risk if failing | Remediation |
|-------|----------------|-------------|
| Visibility correct | High | Verify repo should be public/private |
| Default branch is `main` | Low | `gh api repos/OWNER/REPO --method PATCH -f default_branch=main` |
| Deploy keys | Medium | Review and remove unused keys |
| Actions enabled | Info | Verify allowed actions scope |
| Description present | Low | Add repo description |

### Public Repos Only (403 for Free private)
| Check | Risk if failing | Remediation |
|-------|----------------|-------------|
| Branch protection | Medium | Add ruleset when going public |
| Dependabot alerts | High | Enable and address alerts |

## Risk Levels

| Level | Criteria |
|-------|----------|
| Low | No concerning findings |
| Medium | Missing branch protection (public), deploy keys present |
| High | Open Dependabot alerts (critical/high severity) |
| Critical | Public repo with secrets, excessive deploy keys |

## Practices

### Git Operations
- Never commit secrets, tokens, or credentials
- Never force push to main/master
- Use SSH for remote access (avoid credential helper issues)
- Pin GitHub Actions by SHA, not tag

### CI/CD
- Use `GITHUB_TOKEN` with minimum required permissions
- Pin actions by full SHA hash
- Scope `permissions:` to least privilege
- No `pull_request_target` without careful review

### Dependencies
- Renovate handles dependency update PRs (see [dependency-management.md](dependency-management.md))
- Dependabot handles vulnerability alerts (separate concern — see below)
- Review lockfile changes in PRs
- Prefer well-known registries

### Dependabot Alerts vs Renovate

These tools serve different purposes and both should be active:

| Concern | Tool | How configured |
|---------|------|----------------|
| Dependency update PRs | Renovate (Mend) | `renovate.json` in repo root |
| Vulnerability alerts | Dependabot | Repo settings (not a config file) |

No `dependabot.yml` is needed. Dependabot alerts use the dependency graph automatically — they scan the repo's manifest files (package.json, requirements.txt, Gemfile.lock, etc.) without any per-repo config.

Dependabot alerts are free for all repos (public and private) since 2022. Private repos require manual enablement in Settings > Code security and analysis.

### Access Control
- Principle of least privilege for collaborators
- Review deploy keys periodically
- No shared credentials — use 1Password or GitHub secrets
- Fine-grained PATs over classic tokens

## Security Alert Triage

### Response SLAs

| Severity | Response time | Action |
|----------|--------------|--------|
| Critical | 48 hours | Fix or mitigate immediately; auto-filed as issue by `security-audit.sh --file-issues` |
| High | 1 week | Prioritize in current milestone; flagged in weekly drift report |
| Medium | 2 weeks | Address in next maintenance cycle |
| Low | Next maintenance | Batch with other low-priority work |

### Dismiss criteria

An alert may be dismissed (with documented reason) when:

- **False positive**: the vulnerable code path is not reachable in this project
- **Dev-only dependency**: not present in production builds or runtime (e.g., test framework, linter)
- **No fix available**: upstream has no patched version; accept risk and document in an issue
- **Tolerable risk**: severity is low, attack surface is minimal, and mitigation is disproportionate

Always record the dismiss reason in the alert's dismissal comment for audit trail.

### Escalation

- `security-audit.sh --file-issues all` — auto-files issues on repos with open critical/high alerts
- Weekly `security-audit.sh all` (via `auto-audit-readonly.yml`) flags high-risk repos in maturity report
- Repos with overdue alerts (past SLA) are highlighted in the security audit summary

### Cross-repo auditing

The `security-audit.sh all` command scans all non-fork repos and:
1. Checks whether Dependabot alerts are enabled
2. Counts open alerts by severity
3. Classifies risk level (high/medium/low)
4. Flags SLA violations for overdue alerts
5. Stores results in `data/repo-inventory.json`

## Commands

```bash
# Audit a single repo
./tools/audit/security-audit.sh <repo-name>

# Audit all (dry-run)
./tools/audit/security-audit.sh --dry-run all

# Force re-audit (skip 24h cache)
./tools/audit/security-audit.sh --force all
```

## Upgrading Security Posture

When a private repo goes public:
1. Run security audit first: `security-audit.sh <repo>`
2. Review and remove any secrets in history
3. Enable Dependabot alerts (see below)
4. Add branch ruleset (GitHub's new branch protection)
5. Re-run audit to verify

### Enabling Dependabot Alerts

```bash
# Check if alerts are enabled (204 = enabled, 404 = disabled)
gh api repos/OWNER/REPO/vulnerability-alerts

# Enable for a single repo
gh api repos/OWNER/REPO/vulnerability-alerts -X PUT

# Batch enable for all non-fork repos
jq -r '.repos[] | select(.isFork != true) | .name' data/repo-inventory.json | while read -r repo; do
  if gh api "repos/arechste/${repo}/vulnerability-alerts" -X PUT 2>/dev/null; then
    echo "Enabled: ${repo}"
  else
    echo "Skipped: ${repo}"
  fi
  sleep 1
done
```

The `security-audit.sh` script checks enablement status automatically and reports repos where alerts are disabled.
