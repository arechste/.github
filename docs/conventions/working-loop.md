# Working Loop

Single canonical contract for how work moves from idea to released state in this repo. Every step has an artifact and a gate to the next step; nothing is invisible. Other surfaces (skills, rules, hooks, scripts) execute pieces of this loop; this doc is the union.

## The loop

```
Plan ─► Document ─► Implement ─► Verify ─► Commit ─► PR ─► Merge ─► Release
```

| # | Step | What happens | Artifact | Gate to next step |
|---|---|---|---|---|
| 1 | **Plan** | Goal stated; approach agreed; trade-offs named. Plan mode for novel work; skipped for known workflows (fix, triage, ship, commit, delegation transitions). | Conversation, or `~/.claude/plans/<slug>.md` for non-trivial work | User approval (`ExitPlanMode`) or implicit agreement on a known workflow |
| 2 | **Document** | Non-obvious *why* recorded in the right doc — same PR as the change. Not "what the code does" (the code does that); the why behind a rule, convention, or decision. | Edits in `docs/conventions/`, `.claude/rules/`, `CLAUDE.md`, or in-line comments only when the why is non-obvious | Future readers find the rationale where they look for it |
| 3 | **Implement** | Feature branch from latest `main`. One issue = one branch = one PR. Worktree by default for parallel sessions (see `standing-policies.md`). Branch-naming form follows `.claude/rules/git-workflow.md` (single-machine in this repo: `<type>/<short-description>-<N>`). | Code on the branch | Lint + type pass locally; pre-commit hooks happy |
| 4 | **Verify** | "How to test" stated explicitly *before* the commit. Tests run locally; CI passes. Verification bar named per `change-verification.md` (`not-applicable`, `pending`, `verified`). | Test output, CI green, named bar in PR body | Bar marked `verified` (or `not-applicable` with reason) |
| 5 | **Commit** | Conventional commit format (`type(scope): description (#N)`) per `commit-format.md`. Identity trailer resolved from `~/.claude/commit-trailer.json` — `Auditor` for this repo. One logical change per commit. | Signed commit on the branch | `git push -u origin HEAD` |
| 6 | **PR** | Review-tier banner per `.claude/rules/review-handoff.md` (Skim, Read, Verify, or Block). PR body uses `Fixes #N` for normal work, `Refs #N` for delegated work (auto-close bypasses delegator verification). | PR open with banner, CI running | CI green; tier-appropriate review |
| 7 | **Merge** | Per `merge-gate.md` (resolves on `mergePolicy` from `repo-inventory.json`). For `mergePolicy=auto` repos (dotclaude, dotfiles, git-organizer), Claude self-merges on CI green; for `mergePolicy=human` (mac-organizer), human merges. Squash merge with `gh pr merge --squash --delete-branch`. Local-state cleanup follows `merge-completion.md`. Escalate only on the three triggers (conflict, significant unexpected scope, in doubt). | Squash commit on `main`, branch deleted everywhere, working tree on `main` | Local state matches remote `main` |
| 8 | **Release** | When a milestone closes, `/dc:ship <bump>` cuts the tag and GitHub release per `semver.md`. Human-approved. Consumers pull a tag for a known-good state. | `vX.Y.Z` tag + release notes + closed milestone | Consumers can adopt the release |

## Properties this contract guarantees

- **`main` is always shippable.** Squash-only + CI gate + conventional commits + no half-merged work means every commit on `main` is a coherent state.
- **Releases are consumable.** Tags mark known-good states. A consumer can pull `v1.30.0` and get a defined behavior.
- **Parallel sessions don't collide.** Worktree per session (`standing-policies.md`); each branch independent. The `PreToolUse` advisory lock catches accidental concurrent edits in the same tree.
- **You can see what's been done.** `gh pr list` shows in-flight; `gh issue list --milestone <v>` shows queued vs. closed; `git log v1.x..main` shows what shipped since the last release.
- **Build on previous work.** Every step has an artifact you can cite. No invisible work, no "I'll fix it next session" without a PR or issue.

## Where `/schedule` fits

`/schedule` (remote routines) is **not part of the in-session loop**. It is a separate surface for hands-off verification *against* the contract. Two sanctioned categories today, one explicitly-open category:

| Category | Status | Examples |
|---|---|---|
| **Timed audits** | Sanctioned | Monthly self-audit (`templates/routine-prompt/monthly-self-audit.md`); recurring drift report; schedule-governance compliance check. Read-only by definition. |
| **Handoff tasks** | Sanctioned | Post-deadline cleanup nudge; scheduled scout report on upstream changes; time-sensitive delegation re-nudge. Single-fire bridges where no in-person session would pick up the work. |
| **Scheduled triage / review** | **Rejected** (#484, 2026-05-03) | Triage and review are in-session work. `/dc:triage` and `/dc:review` cover them with current state, hooks, and machine context; `/loop` covers session-internal recurrence. `/schedule` runs in cloud VMs without those — strictly worse for triage by definition. The category may be reopened only if a concrete need surfaces that the in-session loop demonstrably cannot serve. |

Not legitimate `/schedule` uses: in-session work execution, multi-file writes, branch creation, opening PRs, settings changes. That's what the in-session loop is for.

**Hard gate: never create or modify a routine without an explicit user ask in the current session.** When asked, route through the governance question first ("which category? which identity? canary needed?") before creating. Per `.claude/rules/schedule-governance.md` and `docs/conventions/schedule-governance.md`.

## Why this exists

Audit on 2026-05-02 (`~/.claude/plans/audit-and-tell-me-velvet-biscuit.md`) found three remote routines, two of which bypassed the in-session loop:

| Routine | Verdict | Why |
|---|---|---|
| `weekly-ci-budget-check` | Defensible | Passive monitoring; no plausible session pickup. Predates governance #436. Follow-up: #483. |
| `verify-422-ship-or-renudge` | **Overreach** | Delegation verification is in-session work via `/dc:delegate` or `/dc:triage`, not a scheduled agent. |
| `xhigh-effort-canary-649` | **Clear violation** | Multi-file writes, draft PR, branch creation. Disallowed by `~/dotclaude/docs/reference/routines-policy.md` (no hooks → no commit-trailer / deny-rule enforcement). |

All three were created before governance landed (#436, 2026-04-28). The rule worked: nothing has been scheduled in violation since. But the underlying *habit* persisted — sessions reached for `/schedule` whenever a multi-step task felt long, or whenever a session ended with a "natural follow-up." The fix is not more rules saying "don't schedule." It's a single canonical loop that makes the in-session path obvious. When the loop is solid, `/schedule` collapses to its real job: verifying the contract holds.

## See also

- `docs/conventions/semver.md` — version bump rules and milestone-to-semver mapping
- `docs/conventions/commit-format.md` — conventional commit format and identity trailer
- `docs/conventions/merge-completion.md` — local cleanup after a merge
- `docs/conventions/auto-merge-policy.md` — per-repo merge stance and decision matrix
- `docs/conventions/change-verification.md` — verification bars (`not-applicable`, `pending`, `verified`)
- `docs/conventions/schedule-governance.md` — full `/schedule` governance (caps, retirement, cost)
- `.claude/rules/git-workflow.md` — branch naming, PR workflow, worktree default
- `.claude/rules/review-handoff.md` — PR review-tier banner (Skim, Read, Verify, Block)
- `.claude/rules/schedule-governance.md` — in-session enforcement, hard ask-first gate
- `.claude/rules/standing-policies.md` — worktree-by-default rationale
- `~/dotclaude/docs/reference/routines-policy.md` — runtime safety for routines
- Source audit: `~/.claude/plans/audit-and-tell-me-velvet-biscuit.md`
- Source issues: #436 (governance), #482 (this contract), #483 (budget-check follow-up), #484 (triage/review category — rejected 2026-05-03)
