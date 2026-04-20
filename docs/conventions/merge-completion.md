# Merge Completion

Closing the loop locally after a PR merges. Two concerns: **preventing new orphans** (solved by a flag) and **recovering existing orphans** (solved by a tool).

## Prevention — use `--delete-branch`

`gh pr merge --squash --delete-branch <pr-number>` does three things:

1. Deletes the remote branch
2. Switches the local checkout to the base branch and fast-forwards
3. Force-deletes the local feature branch (equivalent to `git branch -D`)

All three without any wrapper, hook, or follow-up script. The "orphan local branch" failure mode only shows up when `--delete-branch` is omitted.

Verified against `gh version 2.90.0+` (stable since 2025). Dogfooded on this repo across PRs #272, #276, #282.

**Rule**: always pass `--delete-branch`. Without it, you own the cleanup.

## Recovery — `/prune` for legacy orphans

Every repo that predates the `--delete-branch` habit has a pile of orphan local branches. `tools/maintenance/prune-merged-branches.sh` (aka `/prune` skill) iterates them, verifies each against the base via local patch-id detection, and deletes the confirmed-merged ones.

- `--dry-run` default; `--yes` for live; `--confirm-via-api` for zero-false-positive mode
- Logs every deletion to `data/audit-log.json`

Use `/prune` periodically or when `git branch` output gets noisy.

## Audit logging — opt-in via `complete-merge.sh`

Skills that want a per-merge audit entry (`data/audit-log.json`) can call `./tools/lib/complete-merge.sh <branch>` after `gh pr merge`. The wrapper:

1. Verifies the branch is squash-merged via the shared detector
2. Is idempotent against cleanup `gh --delete-branch` already did (switch, pull, branch-D all no-op if already clean)
3. Logs `{branch, base, mergeBase, method}` to the audit file

This is **optional**. Skills that don't need the audit trail can skip it. The wrapper adds value only if you want the log entry.

## Detection algorithm

Shared between `complete-merge.sh` and `prune-merged-branches.sh`. Local-first, no network required.

```bash
# For branch <b> against base <base>:
ahead=$(git rev-list --count "$base..$b")
[[ $ahead -eq 0 ]] && return 1     # identical-tree guard (false-positive prevention)
mb=$(git merge-base "$base" "$b")
synth=$(git commit-tree "$b^{tree}" -p "$mb" -m _)
# '-' prefix in cherry output = patch-id already present upstream
[[ "$(git cherry "$base" "$synth" | head -1)" == -* ]]
```

Uses `git cherry` patch-id equivalence (ignores whitespace + line numbers). Detects squash-merge, rebase-merge, and true-merge variants. Reference: `data/references/source_squash_merge_detection.json`.

Primitive lives in `tools/lib/gh-helpers.sh::branch_is_squash_merged`. Both tools call it.

## Base-branch resolution

Auto-detection order:

1. Explicit `--main <branch>` override (when the caller specifies)
2. `git symbolic-ref --short refs/remotes/origin/HEAD` (local, no network)
3. Fallback: `main`

No network call, no dependency on `gh`. Override is the escape hatch for non-standard repos.

## Non-squash merges (`batch` epic path)

When a skill uses `gh pr merge --merge` (true merge commit, not squash), `complete-merge.sh` is **intentionally not applicable** — the detector validates squash-merges only. In that path, rely on `--delete-branch` plus the skill's own orchestration (see dotclaude `tools/post-merge-cleanup.sh`).

## Edge cases (applies to `/prune` recovery)

| Case | Handling |
|---|---|
| Branch tree identical to base (reset-to-main) | `SKIP` `no-commits-ahead` — never trust cherry's `-` on an empty diff |
| Branch checked out in a linked worktree | `SKIP` `worktree` — parse `git worktree list --porcelain` |
| Current HEAD branch | `SKIP` `current-HEAD` |
| Base branch itself | `SKIP` `base-branch` |
| Upstream still present on remote | `SKIP` `upstream-active` — possibly open PR |
| Dirty tree or in-progress rebase/merge/cherry-pick/bisect | exit 1 immediately; never delete |
| Merged from a deleted fork | Local detector catches it (patch-id doesn't care about fork provenance); `--confirm-via-api` may not find a PR |
| Branch merged via direct push to base (no PR) | Local detector catches it; `--confirm-via-api` will fail |

## Permissions

Both tools have scoped permission entries. `git branch -D` is denied globally so ad-hoc LLM attempts to bypass the wrapper prompt for approval.

```jsonc
// Wherever the tools are invoked from (project settings or global baseline)
{
  "permissions": {
    "allow": [
      "Bash(*/git-organizer/tools/lib/complete-merge.sh *)",
      "Bash(*/git-organizer/tools/maintenance/prune-merged-branches.sh *)"
    ],
    "deny": [
      "Bash(git branch -D:*)"
    ]
  }
}
```

## Routines / `/schedule`

Interactive-only for v1. Hosted VMs for scheduled agents do not load `.claude/rules/*` or hooks, so neither `--delete-branch` habit nor the audit wrapper is enforced there. Any scheduled cleanup path must embed its own safety (see `docs/reference/routines-policy.md`) — explicit follow-up work.

## Model tier

- `/prune` skill: `sonnet` — matches other maintenance skills (dry-run → review → confirm flow)
- `complete-merge.sh` is bash-only; no model cost at runtime

## History

The initial v1 contract (PR #273) required every merge-completing skill to call `complete-merge.sh`. Dogfooding on dotclaude (#571 → #578) showed the premise — that `--delete-branch` leaves orphans — was wrong. Revised here (#281) to: `--delete-branch` handles prevention; the wrapper is an opt-in audit path.
