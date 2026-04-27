# Conventions

Normative standards for git/GitHub workflows, repo structure, security, and
cross-repo collaboration. These are the source of truth — `.claude/rules/`
files are the enforceable distillations consumed by Claude sessions.

## Mirrored from dotclaude

These files exist in both `git-organizer/docs/conventions/` and
`~/dotclaude/docs/conventions/`. **git-organizer is authoritative**; dotclaude's
copy is distributed via chezmoi. Sync direction: edit here → apply to dotclaude.

- `adopting-ci.md`
- `commit-format.md`
- `cross-repo-automation.md`
- `dependency-management.md`
- `forge-abstraction.md`
- `issue-hierarchy.md`
- `label-taxonomy.md`
- `local-ci.md`
- `repo-standards.md`
- `security-baseline.md`
- `semver.md`
- `temp-files.md`
- `workspace-management.md`

## Git workflow

| File | Purpose |
|---|---|
| [commit-format.md](commit-format.md) | Conventional commit spec + trailer resolution |
| [merge-completion.md](merge-completion.md) | Post-merge cleanup contract (`gh pr merge --delete-branch` + `complete-merge.sh`) |
| [auto-merge-policy.md](auto-merge-policy.md) | When auto-merge is appropriate (per-repo stance, per-PR-type matrix, PR-as-gate framing) |
| [audit-log-storage.md](audit-log-storage.md) | Audit log is local-only, gitignored — when to write, who reads, why |
| [workspace-management.md](workspace-management.md) | Worktrees, branch lifecycle, parallel work |
| [semver.md](semver.md) | Semantic versioning bump rules and pre-release tags |
| [release-parity.md](release-parity.md) | CHANGELOG.md ↔ GitHub release alignment |
| [change-verification.md](change-verification.md) | Verification bar levels for merged changes |

## Issue & label management

| File | Purpose |
|---|---|
| [label-taxonomy.md](label-taxonomy.md) | Canonical label set (type/, priority/, status/, agent/) |
| [issue-hierarchy.md](issue-hierarchy.md) | Epic → story → task structure and parent-grouping |
| [project-templates.md](project-templates.md) | GitHub Projects v2 fields, automation, repo→project mapping |

## CI & cost

| File | Purpose |
|---|---|
| [adopting-ci.md](adopting-ci.md) | Decision matrix for adding CI workflows to a repo |
| [local-ci.md](local-ci.md) | Local CI parity strategy (act, pre-commit, dry-runs) |
| [billing-api.md](billing-api.md) | GitHub billing API endpoints and budget monitoring |

## Security & secrets

| File | Purpose |
|---|---|
| [security-baseline.md](security-baseline.md) | Minimum security controls per repo class |
| [git-review-checks.md](git-review-checks.md) | PR review checks beyond CI (sensitive paths, scope) |

## Cross-repo & delegation

| File | Purpose |
|---|---|
| [cross-repo-automation.md](cross-repo-automation.md) | Patterns for repos that automate other repos |
| [cross-repo-read-paths.md](cross-repo-read-paths.md) | Readable filesystem paths from this repo |
| [delegation-protocol.md](delegation-protocol.md) | Inbound/outbound delegation lifecycle and labels |
| [fleet-sync.md](fleet-sync.md) | Chezmoi-distributed repo propagation across machines |

## CLI execution

| File | Purpose |
|---|---|
| [claude-cli-execution.md](claude-cli-execution.md) | Full reference for git/gh/Claude CLI patterns |
| [claude-cli-execution-rule.md](claude-cli-execution-rule.md) | Distributable rule (concise) for consuming repos |

## Misc

| File | Purpose |
|---|---|
| [repo-standards.md](repo-standards.md) | Required files, structure, metadata per repo |
| [forge-abstraction.md](forge-abstraction.md) | Forge-agnostic patterns for future GitLab/Gitea support |
| [dependency-management.md](dependency-management.md) | Renovate config, lockfile review, version pinning |
| [github-features.md](github-features.md) | What we use, what's blocked by free-tier, what's planned |
| [permission-matrix.md](permission-matrix.md) | Autonomy level → action matrix |
| [temp-files.md](temp-files.md) | `.tmp/` discipline and naming |

## See also

- [`../guides/`](../guides/) — narrative how-tos
- [`../checklists/`](../checklists/) — review checklists
- [`../../.claude/rules/`](../../.claude/rules/) — rules derived from these conventions
