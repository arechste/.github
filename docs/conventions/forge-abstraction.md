# Forge Abstraction

Maps universal git concepts to provider-specific implementations. git-organizer works across forges by abstracting at the convention level while keeping tooling provider-specific.

## Forge Roadmap

| Phase | Provider | Scope | Status |
|-------|----------|-------|--------|
| 1 | github.com/arechste | 155 repos, full management | Active |
| 2 | github.com/ntnxlab-ch | Collaborative org, peers may lack tooling | Planned |
| 3 | gitlab.com/arechste | Personal GitLab repos | Planned |
| 4 | Self-hosted GitLab/Gitea | On-prem instances | Future |

## Universal Concepts

These concepts exist across all forges. Conventions should reference the generic term; tools implement the provider-specific version.

| Generic Concept | GitHub | GitLab | Gitea/Forgejo |
|----------------|--------|--------|---------------|
| Repository | repository | project | repository |
| Merge request | pull request (PR) | merge request (MR) | pull request |
| CI pipeline | GitHub Actions | GitLab CI/CD | Gitea Actions |
| CI config | `.github/workflows/*.yml` | `.gitlab-ci.yml` | `.gitea/workflows/*.yml` |
| Labels | labels | labels | labels |
| Issue tracker | issues | issues | issues |
| Protected branches | branch protection rules | protected branches | branch protection |
| Merge strategy | squash, merge, rebase | squash, merge, rebase, fast-forward | squash, merge, rebase |
| Forge CLI | `gh` | `glab` | `tea` |
| API style | REST + GraphQL | REST + GraphQL | REST |
| Auth token | PAT (fine-grained) | PAT (project/group) | PAT (basic) |
| CI secrets | repository/org secrets | CI/CD variables | secrets |
| Code review | review + approve | review + approve | review + approve |
| Auto-merge | enabled per-PR | merge when pipeline succeeds | — |
| Webhooks | repository webhooks | project hooks | webhooks |

## CLI Tool Mapping

| Operation | `gh` (GitHub) | `glab` (GitLab) | `tea` (Gitea) |
|-----------|--------------|-----------------|---------------|
| List repos | `gh repo list` | `glab repo list` | `tea repo ls` |
| Create PR/MR | `gh pr create` | `glab mr create` | `tea pr create` |
| API call | `gh api <path>` | `glab api <path>` | `tea api <path>` |
| List labels | `gh label list` | `glab label list` | `tea label ls` |
| Auth status | `gh auth status` | `glab auth status` | `tea login list` |

## API Patterns

### Pagination
- **GitHub**: `--paginate` flag on `gh api`, Link header for raw REST
- **GitLab**: `per_page` + `page` params, `X-Next-Page` header
- **Gitea**: `limit` + `page` params, `X-Total-Count` header

### Rate Limits
- **GitHub**: 5000 req/hr (authenticated), `X-RateLimit-Remaining` header
- **GitLab**: varies by tier, `RateLimit-Remaining` header
- **Gitea**: configurable per-instance, no standard header

### Server-Side Filtering
- **GitHub**: `--jq` on `gh api`, GraphQL for complex queries
- **GitLab**: query params (`state`, `labels`, `search`), GraphQL
- **Gitea**: query params only, no GraphQL

## Convention Applicability

| Convention | All Forges | GitHub-Specific | Notes |
|-----------|-----------|-----------------|-------|
| Conventional commits | Yes | — | Git-level, forge-independent |
| Branch naming (`feat/`, `fix/`) | Yes | — | Git-level |
| Squash merge | Yes | PR settings | Config differs per forge |
| Label taxonomy | Yes | Sync via `gh label` | CLI differs per forge |
| CI checks before merge | Yes | `required_status_checks` | Config differs per forge |
| Signed commits | Yes | Vigilant mode | Verification differs |
| Issue templates | Yes | `.github/ISSUE_TEMPLATE/` | Path differs per forge |
| PR templates | Yes | `.github/pull_request_template.md` | Path differs per forge |
| Dependency scanning | Provider-dependent | Dependabot | GitLab has built-in |

## Implementation Notes

### Current State (Phase 1)

All scripts use `gh` directly. The forge abstraction is at the convention and schema level only — no runtime dispatch layer exists yet.

When Phase 2 begins (ntnxlab-ch), evaluate whether a `forge-helpers.sh` dispatch layer is needed or whether separate scripts per provider are simpler.

### Schema Support

- `data/schemas/forge-provider.schema.json` — provider configuration
- `data/schemas/repo-inventory.schema.json` — `forgeProvider` field on each repo
- `data/references/source_*.json` — `provider` field links knowledge to specific forges
