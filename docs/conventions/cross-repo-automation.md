# Cross-Repo Automation

How git-organizer automates operations across all managed repos.

## Auth Strategy

The default `GITHUB_TOKEN` in GitHub Actions is scoped to the repo running the workflow. Cross-repo writes (label sync, template sync, issue creation) require additional auth.

### Current: Fine-Grained PAT

A fine-grained Personal Access Token (PAT) stored as a repo secret.

**Setup steps** (manual — human must do these):

1. Go to [GitHub Settings > Developer Settings > Fine-grained tokens](https://github.com/settings/tokens?type=beta)
2. Create token with:
   - Name: `git-organizer-automation`
   - Expiration: 90 days
   - Resource owner: `arechste`
   - Repository access: All repositories
   - Permissions:
     - Contents: Read and write
     - Issues: Read and write
     - Metadata: Read-only
3. Copy the token
4. Go to git-organizer repo > Settings > Secrets and variables > Actions
5. Add secret: `CROSS_REPO_PAT` with the token value

**Rotation**: Token expires every 90 days. GitHub sends email reminders. Regenerate and update the secret.

### Future: GitHub App

When managing multiple orgs (e.g., adding ntnxlab-ch), migrate to a GitHub App:
- Auto-rotating tokens (no manual rotation)
- Per-repo installation with granular permissions
- Audit trail for all operations
- No personal account dependency

Migration path: replace `CROSS_REPO_PAT` with App installation token in workflows. Scripts don't change — they use `GH_TOKEN` env var regardless.

## Workflow Integration

The `scheduled-audit.yml` workflow uses the PAT when available:

```yaml
- name: Cross-repo label sync
  if: env.CROSS_REPO_PAT != ''
  env:
    GH_TOKEN: ${{ secrets.CROSS_REPO_PAT }}
  run: ./tools/labels/sync-labels.sh --category active all
```

When no PAT is configured, cross-repo steps are skipped and audits run read-only.

## Automated Operations

| Operation | Auth Required | Automated |
|-----------|---------------|-----------|
| Fetch repo inventory | GITHUB_TOKEN | Yes (weekly) |
| Label audit (read-only) | GITHUB_TOKEN | Yes (weekly) |
| Security audit (read-only) | GITHUB_TOKEN | Yes (weekly) |
| Template drift (read-only) | GITHUB_TOKEN | Yes (weekly) |
| Maturity scoring | GITHUB_TOKEN | Yes (weekly) |
| Label sync (write) | CROSS_REPO_PAT | When PAT configured |
| Template sync (write) | CROSS_REPO_PAT | When PAT configured |
| File issues (write) | CROSS_REPO_PAT | When PAT configured |

## Manual Operations

These remain manual (run from local machine):
- `sync-labels.sh` — ad-hoc label sync
- `sync-github-templates.sh` — ad-hoc template push
- PAT creation and rotation
- Repo category/tag changes in inventory
