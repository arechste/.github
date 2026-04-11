# Adopting CI Standards

Add convention enforcement to your repo by calling git-organizer's reusable workflow.

## Quick Start

Create `.github/workflows/conventions.yml` in your repo:

```yaml
name: Conventions
on:
  pull_request:
    types: [opened, edited, synchronize, labeled, unlabeled]

jobs:
  check:
    uses: arechste/.github/.github/workflows/convention-check.yml@main
    with:
      check-pr-title: true
      check-commits: true
      require-labels: false
```

> **Note:** The workflow lives in the public `arechste/.github` repo — it is mirrored there from `arechste/git-organizer/.github/workflows/` by `tools/maintenance/sync-github-templates.sh`. See [Hosting reusable workflows](#hosting-reusable-workflows) for why.

## What It Checks

### PR Title (default: enabled)

Validates the PR title follows conventional commit format:

```
type(scope): description
```

Valid types: `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `ci`, `perf`, `style`, `build`, `revert`, `wip`

Examples:
- `feat(auth): add login endpoint`
- `fix: resolve null pointer in parser`
- `docs(readme): update installation steps`

### Commit Messages (default: enabled)

Checks each commit message in the PR. Non-conventional commits are **warned** but don't fail the check — since we use squash merge, the PR title becomes the final commit message.

### Required Labels (default: disabled)

When enabled, requires at least one `type/*` label on the PR. Enable this after syncing canonical labels to your repo:

```yaml
require-labels: true
```

## Prerequisites

For label checks to work, sync canonical labels first:

```bash
# From git-organizer checkout
./tools/labels/sync-labels.sh your-repo-name
```

## Inputs

| Input | Type | Default | Description |
|-------|------|---------|-------------|
| `check-pr-title` | boolean | `true` | Validate PR title format |
| `check-commits` | boolean | `true` | Validate commit messages |
| `require-labels` | boolean | `false` | Require `type/*` label on PR |

## Hosting reusable workflows

This section is for maintainers of git-organizer (or any repo that *publishes* reusable workflows), not consumers. Consumers only need the `uses:` line above.

### Use a public repo as the distribution channel

Reusable workflows are kept in `arechste/git-organizer/.github/workflows/` as the single source of truth, then mirrored to the public `arechste/.github` repo by `tools/maintenance/sync-github-templates.sh`. Consumers reference the public mirror. This is the only reliable sharing path on the GitHub free tier — see below.

### Why not call them directly from git-organizer?

`arechste/git-organizer` is private. Private-to-private reusable workflow sharing requires setting the caller repo's `actions/permissions/access` `access_level` to `"user"` (so other repos in the same user namespace can call it). This was discovered after `dotclaude`'s `conventions.yml` silently failed with zero jobs — the reusable workflow call was denied because `access_level` was `"none"` on git-organizer.

Setting `access_level: "user"` is **necessary but not sufficient** on the free tier. Empirically, even with:

- `access_level=user` on the provider (git-organizer) ✓
- `allowed_actions=all` on the consumer ✓
- The workflow pinned by SHA ✓
- The provider workflow not disabled ✓

…private-to-private calls still fail with `startup_failure`. Publishing to a public repo bypasses the restriction entirely.

### Checklist for adding a reusable workflow

1. Add the workflow file to `git-organizer/.github/workflows/<name>.yml` with `on: workflow_call`.
2. Add `<name>.yml` to the `PUBLISHED_WORKFLOWS` allowlist in `tools/maintenance/sync-github-templates.sh`.
3. Run `./tools/maintenance/sync-github-templates.sh --dry-run`, review, then without `--dry-run`.
4. Tag a release on `arechste/.github` so consumers can pin by version (`@v1.2.3`).
5. Keep `access_level: "user"` set on git-organizer as defense-in-depth for internal `uses: ./.github/workflows/...` references — do not set it to `"none"`.

### Verifying access_level on a private provider repo

```bash
gh api repos/arechste/git-organizer/actions/permissions/access --cache 5m
# Expected: {"access_level":"user"}

# To set it (if it drifts back to "none"):
gh api --method PUT repos/arechste/git-organizer/actions/permissions/access \
  -f access_level=user
```

Public repos return HTTP 422 (`Access policy only applies to internal and private repositories`) — this check is private-only.
