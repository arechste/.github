# Merge Gate

Single source of truth for who clicks merge. Every other doc cites this one.

## Two approval points per release

```
plan/issue start  →  Claude executes  →  CI green  →  Claude merges  →  /dc:ship
      ▲                                                                     ▲
      │                                                                     │
   approval 1                                                          approval 2
   (durable: one nod per issue/plan,                              (closing review:
    not per PR)                                                     PR-summary view)
```

**Between** the two: no per-PR human gate. Claude executes, verifies, merges.

## Escalation triggers (rare, narrow)

Pause and ask **one targeted question** when any of these fire:

1. **Merge conflict** — `gh pr merge` would refuse; rebase/manual resolution needed
2. **Significant unexpected scope change** — work grew substantially past the issue's acceptance or the approved plan
3. **In doubt** — Claude self-attests low confidence in the change, the verification, or any other aspect

These three. Not a longer list. Sensitive-path edits, large diffs, or convention ripples are not separate triggers — if they were already in the plan, they're authorized; if they weren't, they fall under "scope change" or "in doubt."

## Repo bindings

Source: `dotclaude/data/repo-inventory.json` → `repos[].mergePolicy`.

| Repo | `mergePolicy` | Effect |
|---|---|---|
| `dotclaude` | `auto` | Claude self-merges on CI green |
| `dotfiles` | `auto` | Claude self-merges on CI green |
| `git-organizer` | `auto` | Claude self-merges on CI green |
| `mac-organizer` | `human` | Human-only (boot-critical fleet — explicit override) |

`mergePolicy=auto` is the default for low-blast-radius repos. `mergePolicy=human` is the explicit override for repos where revert is operationally expensive (machines already touched).

A third value `mergePolicy=conditional` may be set by a repo that wants the per-PR matrix in `auto-merge-policy.md` (chore/docs/test/ci → auto; feat/refactor/security → human; fix → priority-keyed). Not used by any team repo today.

## Release gate

Source: `~/dotclaude/home/skills/ship/SKILL.md` step 7.

`/dc:ship` runs autonomously through changelog generation, branch, PR, CI poll, and merge. **Then it pauses** at `AskUserQuestion` with:

- The release version + last-tag delta
- Changelog (commit messages, grouped by type)
- PR-summary block: each merged PR in the milestone with its branch type, verification bar, and one-line title

User answers `yes` → Claude tags, creates the GitHub release, closes the milestone, opens the next one. `no` → release branch stays open for manual handling.

This is the **only** scheduled human approval point in the release cycle. Per-PR review still happens (review-tier banner in PR body), but it does not gate merge for `auto` repos.

## What this replaces

This doc supersedes the merge-gating prose previously scattered across:

- `permission-matrix.md` (action matrix rows for `gh pr merge`)
- `review-handoff.md` (Skim/Read/Verify/Block merge column)
- `agent-modes.md` ("Human always merges")
- `auto-merge-policy.md` (Path 3 / per-PR matrix for `conditional`)
- `working-loop.md` row 7 (Merge step)

Those docs now cite this one. The per-PR matrix in `auto-merge-policy.md` remains canonical for `mergePolicy=conditional` repos but does not gate `auto` repos.

## See also

- `docs/conventions/working-loop.md` — full Plan → Release loop
- `docs/conventions/change-verification.md` — verification bar named in every PR
- `docs/conventions/auto-merge-policy.md` — per-PR matrix for `conditional` repos
- `.claude/rules/review-handoff.md` — review-tier banner format (still required in every PR body for review value, even when not merge-gating)
- `.claude/rules/triage-cadence.md` — issue-to-milestone routing
