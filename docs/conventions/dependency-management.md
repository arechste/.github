# Dependency Management Convention

How dependencies are tracked, updated, and governed across managed repos.

## Decision: Renovate

**Chosen**: Mend Renovate (GitHub App)
**Over**: GitHub Dependabot (native)
**When**: 2026-03-26

### Rationale

| Factor | Renovate | Dependabot |
|--------|----------|-----------|
| Package managers | 90+ (Brewfile, pre-commit, Actions, Docker Compose, Helm) | ~30 (no Brewfile, limited Docker) |
| Multi-forge | GitHub, GitLab, Bitbucket, Azure DevOps, Gitea | GitHub only |
| Shared presets | `local>owner/repo` — centralized config across repos | Per-repo config only |
| Scheduling | Cron-like, rate limiting, concurrency control | Basic scheduling |
| Conventional commits | Built-in (`chore(deps):` default) | Not native |
| Automerge | Native with configurable policies | Requires separate Actions workflow |
| Dependency dashboard | Built-in issue for tracking all pending updates | No equivalent |
| CI budget control | `commitHourlyLimit`, `prConcurrentLimit`, `prHourlyLimit` | `open-pull-requests-limit` only |
| Cost | Free (GitHub App runs on Mend infra) | Free (GitHub native) |

Key deciding factors:
1. **Multi-forge roadmap** — we plan to expand to GitLab/self-hosted; Renovate works there, Dependabot doesn't
2. **Brewfile support** — dotfiles uses Homebrew for tool management; Dependabot can't update it
3. **Shared presets** — one config in git-organizer, extended by 155 repos without duplication
4. **CI budget** — fine-grained rate limiting prevents exhausting GitHub Free tier minutes

### What Renovate does NOT replace

- **Dependabot alerts** — keep enabled on all repos (free vulnerability scanning via repo settings, not config files). See [security-baseline.md](security-baseline.md) for triage SLAs and enablement
- **`gh-best-practices.sh`** — Renovate updates deps, it doesn't audit Actions security patterns
- **Pre-commit hooks** — Renovate can update hook versions, but pre-commit itself runs locally

## Dependency Surfaces

| Surface | Manager | Where used |
|---------|---------|-----------|
| GitHub Actions | `github-actions` | All repos with CI workflows |
| Pre-commit hooks | `pre-commit` | git-organizer, dotclaude, dotfiles |
| Docker images | `docker-compose`, `dockerfile` | Repos using containers, .actrc |
| Homebrew | `homebrew` | dotfiles (Brewfile) |
| npm/pip/go | respective managers | Active repos with runtime deps |

## Shared Preset

Stored at `renovate/default.json` in this repo. Downstream repos extend it:

```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["local>arechste/git-organizer"]
}
```

### Preset contents

The shared preset enforces:
- **Schedule**: weekly, after 10pm (avoids CI spikes during work hours)
- **Conventional commits**: `chore(deps):` prefix (matches our git conventions)
- **Grouping**: minor + patch updates grouped per manager to reduce PR noise
- **Actions pinning**: SHA pinning with version comments
- **Pre-commit**: enabled (disabled by default in Renovate)
- **Rate limiting**: max 2 commits/hr, max 5 concurrent PRs
- **Automerge**: patch-level dev deps only (low risk)
- **Labels**: `type/chore`, `priority/low` (matches our label taxonomy)

## Repo Targeting

Not every repo needs Renovate. Target by category:

| Category | Renovate? | Rationale |
|----------|-----------|-----------|
| **active** | Yes | Has dependencies worth tracking |
| **infrastructure** | Yes | CI/tools need version governance |
| **learning** | Optional | Low priority; enable if actively maintained |
| **dormant** | No | No active development |
| **fork** | No | Upstream manages dependencies |
| **profile** | No | No meaningful dependencies |

### Enabling for a repo

1. Install the Renovate GitHub App on the repo (or enable for the org)
2. Add `renovate.json` to the repo root:
   ```json
   {
     "$schema": "https://docs.renovatebot.com/renovate-schema.json",
     "extends": ["local>arechste/git-organizer"]
   }
   ```
3. Renovate creates a Dependency Dashboard issue on first run
4. Review and merge onboarding PR

### Per-repo overrides

Repos can override preset settings in their `renovate.json`:

```json
{
  "extends": ["local>arechste/git-organizer"],
  "schedule": ["before 7am on Monday"],
  "automerge": false
}
```

## CI Budget Considerations

GitHub Free tier: 2000 Actions minutes/month for private repos.

### How Renovate affects CI

- Renovate itself runs on Mend's infrastructure (zero Actions minutes)
- Each Renovate PR triggers CI workflows when created/rebased
- With rate limiting: max 2 PR creates/hr, max 5 concurrent PRs

### Budget math (worst case)

- 10 active private repos, each gets ~5 Renovate PRs/week
- Average CI run: 3 minutes
- Weekly Renovate CI cost: 10 x 5 x 3 = 150 minutes
- Monthly: ~600 minutes (~30% of budget)

### Mitigation

- Schedule updates after 10pm to spread CI load
- Use `commitHourlyLimit: 2` to prevent bursts
- Group minor+patch updates to reduce PR count
- Automerge low-risk updates (fewer rebase cycles)
- Keep Renovate disabled on dormant/fork repos

## Rollout Plan

### Phase 1: Pilot (git-organizer)

- Create shared preset in this repo
- Enable Renovate on git-organizer itself
- Validate: pre-commit updates, Actions SHA updates, .actrc Docker image
- Tune rate limits based on observed CI impact

### Phase 2: Core repos (dotclaude, dotfiles)

- Enable on dotclaude (pre-commit, Actions)
- Enable on dotfiles (Brewfile, pre-commit, Actions)
- Validate Brewfile manager works correctly

### Phase 3: Active repos

- Enable on remaining `active` category repos
- Monitor CI minute consumption via `local-check.sh --usage`
- Adjust scheduling/limits if budget is tight

### Phase 4: Infrastructure repos

- Enable on `infrastructure` repos
- These typically have simpler dependency surfaces

## Health Check Integration

`data/repo-health-checks.json` includes a `deps-renovate-config` check:
- **Severity**: recommended (for active/infrastructure repos)
- **Check**: repo has `renovate.json` extending shared preset
- Consumed by `dc:repo-health` skill

## Revisit Criteria

Re-evaluate this decision if:
- Dependabot adds Brewfile support and shared config presets
- Renovate's free tier adds limitations
- Multi-forge migration is abandoned (removes key Renovate advantage)
- CI budget becomes unmanageable despite rate limiting

## Sources

- [Renovate Documentation](https://docs.renovatebot.com/)
- [Renovate vs Dependabot](https://docs.renovatebot.com/bot-comparison/)
- [Renovate Shareable Config Presets](https://docs.renovatebot.com/config-presets/)
- [Renovate Scheduling](https://docs.renovatebot.com/key-concepts/scheduling/)
- [GitHub Dependabot Documentation](https://docs.github.com/en/code-security/dependabot)
