# GitHub Features Inventory

> **TL;DR**: We use Issues, Actions, Labels, and Milestones heavily. Branch protection requires Pro. Renovate not needed here.

## Feature status board

```
 Feature                    Status
 ══════════════════════════════════════════════════════════════════════════
 REPOSITORY
 ──────────────────────────────────────────────────────────────────────────
 Issues                     [████] Used — conventional titles, 3 label groups
 Issue templates             [████] Used — bug.yml + feature.yml (YAML forms)
 PR template                [████] Used — checklist in .github/
 Labels                     [████] Used — 30 canonical, synced via scripts
 Milestones                 [████] Used — map to semver (v0.2.0 - v1.1.0)
 Projects v2                [██░░] Documented — board fields defined, not automated
 Releases                   [██░░] Partial — tags exist, no release notes yet
 Discussions                [░░░░] Not planned — single-user project
 Wiki                       [░░░░] Not planned — docs live in repo under VCS
 Pages                      [░░░░] Not planned — no public docs site needed
 ──────────────────────────────────────────────────────────────────────────
 CI / CD
 ──────────────────────────────────────────────────────────────────────────
 Actions workflows          [████] Used — 5 workflows
 Reusable workflows         [████] Used — convention-check.yml (workflow_call)
 Scheduled workflows        [████] Used — Monday 9am UTC weekly audit
 workflow_dispatch           [████] Used — manual trigger on scheduled-audit
 Environments               [░░░░] Not used — no deployment targets
 ──────────────────────────────────────────────────────────────────────────
 SECURITY
 ──────────────────────────────────────────────────────────────────────────
 Dependabot alerts           [██░░] Partial — works on public repos, 403 on Free private
 Dependabot version updates  [░░░░] Not configured — no package manifests
 Secret scanning             [░░░░] Not applicable — Free tier; local hooks cover this
 Code scanning (CodeQL)      [░░░░] Not used — no CodeQL for bash
 Branch protection           [xxxx] Blocked — requires Pro for private repos
 Rulesets                    [xxxx] Blocked — requires Pro for private repos
 Required reviewers          [xxxx] Blocked — requires Pro for private repos
 ──────────────────────────────────────────────────────────────────────────
 COLLABORATION
 ──────────────────────────────────────────────────────────────────────────
 CODEOWNERS                 [░░░░] Not created — should add (* @arechste)
 Autolinks                  [░░░░] Not used — low value for this project
 Saved replies              [░░░░] Not used — could speed up PR review
 ──────────────────────────────────────────────────────────────────────────
 DEPENDENCY MANAGEMENT
 ──────────────────────────────────────────────────────────────────────────
 Renovate                   [░░░░] Not configured — no runtime deps (pure shell)
 GitHub Apps                [░░░░] Future — migration path from PAT when multi-org
 ══════════════════════════════════════════════════════════════════════════

 Legend:  [████] fully used   [██░░] partial   [░░░░] not configured/planned
          [xxxx] blocked (platform limitation)
```

## GitHub Free tier limitations

These features require GitHub Pro, Team, or Enterprise for **private repos**:

| Feature | Tier required | Workaround |
|---------|---------------|------------|
| Branch protection rules | Pro | CI checks + PR discipline |
| Rulesets | Pro | Convention enforcement via CI |
| Required reviewers | Pro | Agent mode rules (human reviews in all modes) |
| Required status checks | Pro | CI runs but can't block merge programmatically |
| `gh pr merge --auto` (native auto-merge) | Pro | Local poll-and-merge helper in `work` skill — see [auto-merge-policy.md](auto-merge-policy.md) |
| Code scanning (private) | GitHub Advanced Security | ShellCheck CI + local hooks |
| Secret scanning (push) | GitHub Advanced Security | Pre-commit hooks via dotfiles |
| Dependabot API (private) | Pro | Public repos only; manual security review for private |

## Recommended additions

| Feature | Why | Effort |
|---------|-----|--------|
| CODEOWNERS | Explicit ownership, maturity score improvement | XS — one file: `* @arechste` |
| GitHub Releases with notes | Tags exist but no release notes on GitHub | S — use `gh release create` with changelog |

## Why not Renovate?

```
 This repo:   tools/*.sh + data/*.json + docs/*.md
              └── no package.json, no requirements.txt, no go.mod
              └── nothing for Renovate to update

 Recommend Renovate for:
   - active repos with package managers
   - infrastructure repos with Terraform/Docker
   - Renovate PRs pass the same convention-check CI as manual PRs
   - Add renovate.json to repos via template sync when ready
```

## Planning features

### Milestones

```
 Two milestones always open: one version milestone + permanent Backlog.
 Triage promotes issues from Backlog → version milestone.
 Milestone close triggers a minor release (see semver.md).

 History: v0.2.0 through v1.16.0 (19 milestones closed).
 Current: v1.17.0 (open) + Backlog (permanent).
 Query: gh api repos/arechste/git-organizer/milestones --jq '.[].title'
```

See: [rules/workflow-conventions.md](../rules/workflow-conventions.md#milestone-conventions)

### Issue lifecycle

```
 Created ──> status/ready ──> status/in-progress ──> PR opened ──> PR merged ──> Auto-closed
    │              │                  │                    │              │
    │              │                  │                    │              └── "Fixes #N" closes issue
    │              │                  │                    └── Moves to "In Review" on board
    │              │                  └── Branch exists, work in progress
    │              └── Triaged, requirements clear
    └── type/*, priority/*, agent/* labels applied
```

See: [rules/workflow-conventions.md](../rules/workflow-conventions.md#issue-lifecycle)

### Projects v2

Board structure defined but not fully automated. Fields: Status, Priority, Size.

See: [rules/workflow-conventions.md](../rules/workflow-conventions.md#github-projects-v2)
