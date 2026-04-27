# Post-Merge Change Verification

A merged PR is not a verified change. Convention docs ship without adoption, skills merge without running, tools land without being exercised on a target, chezmoi-distributed updates reach one machine and stall. This convention defines the **minimum bar** for considering a change actually in effect, by change type.

**This convention is guidance, not a CI gate.** It exists so skills and humans know what "done" means past the merge.

## Bar by change type

| Change type | Verification bar | Automatable |
|-------------|------------------|-------------|
| Convention doc (`docs/conventions/*.md`) | Cited in ≥1 skill, rule, or CI check within 30 days; if unreferenced, surfaces in `dc:health` | Yes |
| Skill (`.claude/skills/*/SKILL.md`, `~/dotclaude/home/skills/*/SKILL.md`) | One dry-run invocation against the intended repo within 24h of merge; outcome recorded (issue comment or PR comment) | Partial |
| Tool (`tools/**/*.sh`) | Runs clean on one target post-merge with visible output (PR comment, issue comment, or commit), or the tool is declared read-only | Yes |
| Label / template change | Dry-run `label-sync` / `template-sync` against ≥1 active repo shows the expected diff | Yes |
| Chezmoi-distributed (dotclaude, dotfiles) | Author runs `chezmoi apply` on the primary machine before merge; a tracking note for other machines is filed (issue comment or fleet-sync log) | Partial |

### Rationale

The "merge-then-retweak" failure is the predictable case: convention ships, no repo adopts it; skill ships, another machine doesn't `chezmoi apply`; tool ships, no observable run on a real target. Each row turns the implicit success signal into an explicit one.

The bar is deliberately **low** — not test coverage, not broad rollout — because blocking every merge on proof-of-adoption is too slow for this project's cadence. The 30-day unreferenced check is a long-horizon safety net; the 24h skill dry-run is a short one.

## What "verified" looks like

- **Convention doc**: a grep from any consuming repo finds the file name as an `@import` in a skill, or as a referenced path in a rule or audit workflow.
- **Skill**: a `gh issue comment` linking to the dry-run outcome, or the skill produces output that clearly shows it ran.
- **Tool**: a PR comment, issue comment, or commit on a real target with a `timestamp` after the merge `SHA`.
- **Label/template**: a subsequent PR or comment showing the diff matched the author's intent.
- **Chezmoi-distributed**: the author confirms `chezmoi apply` on their primary machine; other machines are tracked in the fleet-sync log (see `docs/conventions/fleet-sync.md`).

## Skill integration

Skills that close a `status/review-ready → done` lifecycle (`/dc:ship`, `/dc:work`, `/dc:batch`) emit a **Verification done?** line in their final report. Format:

```
Verification done?
  - <change type>: <bar>
  - status: <not-applicable | pending | verified>
  - next: <command or checklist item the author should run>
```

- `not-applicable` — this change type has no bar (e.g., a trivial doc typo).
- `pending` — the change is merged but the bar is not yet met; the skill prints the commands the author should run.
- `verified` — the skill ran the check itself and it passed.

The skill is not required to block on `pending`. The goal is **visibility**, not enforcement.

## Failure mode: stale convention docs

If a convention doc (`docs/conventions/*.md`) has no incoming reference after 30 days, `dc:health` (or an equivalent audit) surfaces it. Options:

1. Cite it from a skill, rule, or checklist that applies.
2. Delete it — if nothing consumes it, the doc is not load-bearing and should not accrue maintenance cost.

This is guidance, not automated deletion. The audit surfaces the drift; the author decides.

## What this convention is not

- Not a CI-blocking check. No merge is gated on this.
- Not a replacement for tests or CI. Verification here is about **deployment and adoption**, not correctness.
- Not a substitute for `complete-merge.sh`. That tool logs the merge event; this convention verifies the change took effect.

## References

- Audit: `docs/audit/2026-04-18-git-workflow-audit.md` § Axis 4, Inconsistency #7
- Related: `docs/conventions/merge-completion.md` (closing the merge loop locally)
- Related: `docs/conventions/fleet-sync.md` (chezmoi propagation)
- Related: `docs/conventions/delegation-protocol.md` (only place with a pre-existing verification gate)
