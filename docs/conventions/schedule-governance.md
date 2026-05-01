# Schedule Governance

How `/schedule` (remote agents on cron triggers) and `/loop` (in-session recurring prompts) are governed in this project. Covers identity ownership, scope, inventory, retirement, cost, and the offer-pattern.

## Status

**Active** — scope: `arechste/git-organizer`. Cross-references the runtime policy in `~/dotclaude/docs/reference/routines-policy.md` (which covers *what is safe to run*); this document covers *which schedules exist, who owns them, and when to add or retire them*.

## TL;DR

- Default in this project: do NOT proactively offer `/schedule`. File an issue instead.
- Routine identity is `Routines@<hostname>` — distinct from session identities (`Auditor`, `Builder`, `Ops`).
- Inventory is `gh routines list` (via `/dc:schedule list`); routine prompts checked into the repo live under `templates/routine-prompt/`.
- One-shot routines retire on first fire; recurring routines retire after 3 cycles of no actionable output.
- `/schedule` runs in Anthropic-hosted VMs — separate budget from local-first CI.

## Q1 — Identities × scopes

### Routine identity

Routines run in Anthropic-hosted cloud VMs. They do not load `~/.claude/commit-trailer.json` and have no per-machine context. The canonical routine identity is:

```
Co-Authored-By: Claude/<model>/Routines@<hostname> <noreply@anthropic.com>
```

`<hostname>` is the machine where the routine was authored — it identifies who scheduled the routine, not where it ran. Today that is typically `merktnix`. Per the runtime policy, routines must embed this trailer inline because the commit-trailer hook does not fire in the cloud VM.

### Session identities (for context only)

Session identities map per-repo via `~/.claude/commit-trailer.json`:

| Repo | Identity |
|---|---|
| git-organizer | `Auditor` |
| dotclaude | `Builder` |
| dotfiles | `Ops` |

Routines never use these — `Routines@<hostname>` is a separate dimension that signals "this commit was authored by a scheduled agent, not an interactive session."

### Repo-of-record

Every routine prompt MUST encode the repo-of-record explicitly in the `CONTEXT` section. Example: `templates/routine-prompt/monthly-self-audit.md` opens with `for the arechste/git-organizer repository`. A routine without a stated repo-of-record cannot pass review.

Cross-repo routines (e.g., a fleet-wide audit that touches multiple repos) MUST name a single owner repo and a single owner identity. Use the repo whose CLAUDE.md the routine's PRs would be reviewed against.

## Q2 — Inventory & retirement

### Canonical list

The only authoritative inventory of running routines is the claude.ai trigger list:

```bash
/dc:schedule list
# or, programmatically:
# RemoteTrigger action="list"
```

There is no project-local mirror — by design, the live state is the source of truth. This document does not enumerate active triggers.

### Checked-in routine prompts

Routine prompts that have been promoted past the canary stage MUST be checked into `templates/routine-prompt/<slug>.md`. CI lints them via `tools/lint-routine-prompt.sh` (in dotclaude). Anything not in that directory is either (a) a draft / canary, or (b) a one-shot that does not warrant version-controlling.

### Retirement signals

| Routine type | Retirement trigger |
|---|---|
| One-shot (e.g., "open a cleanup PR for flag X in 2 weeks") | Fires once → delete the trigger; the prompt file under `templates/routine-prompt/` may also be removed |
| Recurring (e.g., monthly self-audit) | 3 consecutive cycles produce no actionable output (no PR, no issue, no comment that led to action) → retire or refactor |
| Recurring | Superseded by a hook, a skill, or a manual workflow → retire |
| Recurring | Cost / noise complaint from the user → retire on the next cycle |

### Review cadence

The Pillar 5 monthly self-audit (`templates/routine-prompt/monthly-self-audit.md`, issue #414) reads `gh routines list` as part of its signal collection and flags routines hitting any retirement trigger above. There is no separate monthly review for schedules.

## Q3 — Cost & runtime

### Where routines run

| Tool | Runtime | Billing |
|---|---|---|
| `/schedule` (remote trigger) | Anthropic-hosted cloud VM | Anthropic API account |
| `/loop` (local recurring prompt) | Current Claude Code session | Anthropic API account (same as the session) |
| Hooks (`PreToolUse`, `SessionStart`, etc.) | Local shell | Free (no Claude invocation) |
| GitHub Actions workflows | GitHub-hosted runners | GitHub Actions minutes (per `project_ci_cost_strategy`) |

`/schedule` does NOT count toward the GitHub Actions budget tracked by `tools/ci/budget-monitor.sh`. It is a separate cost surface (Anthropic API usage). The local-first CI strategy continues to apply to GitHub Actions; it does not constrain routines.

### Decision matrix: which tool to reach for

| Situation | Tool |
|---|---|
| Follow-up work during an active session, plausibly picked up via `/triage` next session | **File an issue** (default) |
| Repeating local poll inside the current session ("watch this CI run") | `/loop` |
| Genuinely passive recurring work with no plausible session pickup (e.g., monthly self-audit) | `/schedule` |
| Event-driven local action (e.g., session-start machine warning) | **Hook** (`SessionStart`, `PreToolUse`, etc.) |
| Reactive on push / PR / merge | **GitHub Actions workflow** (subject to CI cost strategy) |

The first row is the default. Reach for `/schedule` only when *both* (1) the work is genuinely passive (no session would naturally pick it up) AND (2) the user has explicitly asked OR a documented governance pattern matches.

### Frequency cap

- Default: max **1 recurring routine** per repo unless the convention adds an exception.
- Current exception: `monthly-self-audit` (Pillar 5, issue #414) — the only sanctioned recurring routine for git-organizer.
- One-shot routines have no cap, but each one MUST file an issue first that records the schedule, identity, and expected fire date — so retirement is auditable.

## Q4 — Safety & idempotency

### Runtime safety policy

The runtime policy is owned by dotclaude: [`~/dotclaude/docs/reference/routines-policy.md`](../../../../dotclaude/docs/reference/routines-policy.md). It defines:

- What is missing in a routine session (no hooks, no rules, no `deny` permissions, no commit-trailer injection)
- Allowed task types (read-only triage, nudges, scout reports, draft generation)
- Disallowed task types (file writes, `git push`, credential access, destructive shell ops, auto-merge, settings changes)
- Required pre-flight (mandatory `SAFETY` block in the prompt)
- Validation gate: read-only canary before any write-capable routine ships

This convention does not duplicate that. It only adds project-level governance on top.

### Project-level requirements

In addition to the runtime policy:

1. **Default-read-only.** New routines start read-only. A write-capable routine requires (a) an issue with `type/feature` documenting the case, and (b) a passing canary recorded in that issue.
2. **Linted prompt.** Every checked-in routine prompt under `templates/routine-prompt/` MUST pass `tools/lint-routine-prompt.sh`. CI enforces this.
3. **Identity in trailer.** The prompt's `SAFETY` block MUST include the `Routines@<hostname>` trailer (per Q1).
4. **No auto-merge.** Any PR opened by a routine waits for human review per the project merge policy. The runtime policy already says this; restated here so reviewers can cite a single source for the `--auto` decision.

## Q5 — Discovery & offer-pattern

### Proactive offers in this project: suppressed

The system prompt encourages Claude to proactively offer `/schedule` after work that has a natural future follow-up. **In this project, that default is suppressed.** Issues are the canonical surface for follow-ups during active sessions.

When a follow-up surfaces during a session:

1. **Default**: file an issue with the appropriate milestone and labels. That is the project's tracking surface.
2. **`/schedule` is appropriate only if** the follow-up is genuinely passive — there is no plausible session that will pick it up via `/triage` — AND the user explicitly asks OR a documented governance pattern matches (only `monthly-self-audit` today).
3. **If the user asks about scheduling something**: route to the governance question first ("which routine identity should own this? is the work passive?") rather than creating the trigger immediately.

This rule lives in `.claude/rules/schedule-governance.md` for in-session enforcement.

### When the project would relax this

The "no proactive offers" stance can be lifted if all of the following hold:

- A second sanctioned recurring routine has shipped and proven retirement-friendly.
- `gh routines list` (or equivalent) is integrated into a project audit that flags drift automatically.
- The user explicitly approves a relaxation in a written decision (issue or PR).

Until then, the project default stands.

## References

- Runtime policy: `~/dotclaude/docs/reference/routines-policy.md`
- Prompt template: `~/dotclaude/templates/routine-prompt/TEMPLATE.md`
- Skill: `~/dotclaude/home/skills/schedule/SKILL.md` (`/dc:schedule`)
- Linter: `~/dotclaude/tools/lint-routine-prompt.sh`
- Project rule: `.claude/rules/schedule-governance.md`
- Sanctioned routine: `templates/routine-prompt/monthly-self-audit.md` (Pillar 5, issue #414)
- Session identity map: `~/.claude/commit-trailer.json`
- Operating-model charter: arechste/git-organizer#396 (Pillar 4 — Govern)
- Source issue: arechste/git-organizer#436
