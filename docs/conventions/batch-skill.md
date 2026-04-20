# Batch Skill Convention

Canonical specification for the `/batch` skill. git-organizer defines the convention; dotclaude distributes it as `dc:batch`.

## Purpose

Execute multiple GitHub issues sequentially: claim, branch, implement, PR, gate on CI, merge, and clean up. Bias toward autonomy — preflight is the gate, then the loop runs without per-issue confirmation.

## Modes

The skill accepts three input modes. Exactly one must be used per invocation.

| Mode | Argument | Resolves to |
|------|----------|-------------|
| Explicit | `#N1 #N2 ...` (bare integers) | The listed issues |
| Milestone | `--milestone vX.Y.Z` | All open `agent/claude` issues in the milestone |
| Label | `--label <label>` | All open issues with the label |
| Parent | `--parent N` | All open sub-issues of issue `#N` |

Common flags:

- `--dry-run` — preflight only, exit code 2
- `--stop-on-failure` — halt on first failure (default: continue)

If no eligible issues resolve, report "No eligible issues found" and stop.

## Eligibility

An issue is eligible when ALL apply:

- Open
- Has `agent/claude` label
- Does NOT have `status/blocked` label
- No `machine/<name>` label mismatch against `$(hostname -s)`
- Does NOT have `meta/tracking` label (parents are not work units)

Ineligible issues are listed with a reason in the preflight table and skipped.

## Preflight

1. Resolve `REPO` (`gh repo view --json nameWithOwner`) and `HOSTNAME` (`hostname -s`)
2. Resolve the input mode to a candidate issue list
3. Fetch each candidate via `gh issue view N --json number,title,labels,state,body` and classify eligibility
4. Check 1Password session (warn-only: `op account list`)
5. Render the preflight table: `# | Title | Labels | Eligible | Reason`
6. If `--dry-run`: print table, exit 2

## Execution Loop

For each eligible issue, in order:

1. **Claim**: `gh issue edit N --add-label "status/in-progress"`
2. **Delegate to a subagent** with a self-contained prompt covering: branch creation (3 calls — `git switch main`, `git pull`, `git switch -c <type>/description-N`), implementation, commit (heredoc trailer from `~/.claude/commit-trailer.json`), push, PR body in `.tmp/pr-body-<suffix>.md`, `gh pr checks --watch --fail-fast`, then `gh pr merge --squash --delete-branch` (never `--auto`)
3. **Post-merge cleanup**: `./tools/post-merge-cleanup.sh --issue N --branch <branch>` (if present)
4. **Release claim**: `gh issue edit N --remove-label "status/in-progress"`
5. **Report**: post an issue comment — `Batch: PR #X merged.` or `Batch: failed — <status>.`
6. On failure with `--stop-on-failure`: halt the batch

## Parent Mode

When invoked with `--parent N`:

1. Resolve the parent's sub-issues:
   ```bash
   source tools/lib/gh-helpers.sh
   gh_list_sub_issues "$REPO" "$N"
   ```
2. The parent itself is **never** executed — it is a tracking container (has `meta/tracking`)
3. Apply the same eligibility filter as other modes
4. Execute in **sub-issue order** as returned by the REST API (GitHub preserves creation order; reorder with the sub-issue reprioritization endpoint if needed)
5. Ineligible sub-issues (closed, `status/blocked`, not `agent/claude`) appear in the preflight table with a reason and are skipped

This mode is the batch analog of triage's parent-grouping view: `triage` shows the hierarchy, `batch` executes it.

## Guard Rails

- Only `agent/claude` issues are executed (verified against live GitHub state)
- Never push to main; never force-push
- Escalate to the human on: merge conflicts, scope ambiguity, security-sensitive files
- CI failure: release the claim, post a summary comment, continue (unless `--stop-on-failure`)
- `--auto` merge is forbidden — private repos without branch protection race CI

## Autonomy Level Adaptation

The skill adapts to the repo's `autonomyLevel` in `data/repo-inventory.json`:

| Autonomy | Batch behavior |
|----------|----------------|
| `read-only` | Preflight table only. No claiming, no execution. |
| `assist` | Preflight + suggested commands. User executes. |
| `pr-only` | Full batch. Human merges each PR. |
| `full` | Full batch. Self-merges after CI passes. |

When running outside git-organizer (no inventory), default to `pr-only`.

## Rules

- Do NOT enter plan mode — batch is action-oriented
- Do NOT ask for confirmation between issues — preflight is the gate
- Always run preflight, even for a single issue
- Truncate issue bodies to 2000 chars in subagent prompts
- One mode per invocation — reject combinations of `#N`, `--milestone`, `--label`, `--parent`
