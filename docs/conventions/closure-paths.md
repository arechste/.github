# Closure Paths

Govern pillar (P4 of #396): canonical mapping from drift / security signal class to closure mechanism.

The rule is: every signal that appears in a drift report must have a defined closure path that produces
an **actionable artifact** (PR opened or delegation filed) without manual stitching. Signals that are
not yet wired are listed with their planned path so they remain visible in the backlog.

## Signal → Closure Mechanism

| Signal class | Source | Closure mechanism | Tooling |
|---|---|---|---|
| Label drift | `bulk-label-audit.sh` | Open PR to target repo via `sync-labels.sh` | `govern-closure.sh --signal label-drift` |
| Security baseline finding | `security-audit.sh` | File issue in target repo (severity-graded) | `govern-closure.sh --signal security-baseline` |
| Template drift | `template-drift.sh` | Open PR via `sync-github-templates.sh` | Planned — not yet wired |
| Convention drift (non-owned repo) | `convention-checker.md` agent | File delegation via `delegate-issue.sh` | Planned — not yet wired |
| Convention drift (git-organizer itself) | Internal audit | Open self-PR | Ad hoc today; skill planned |

## Pilot Signals (wired end-to-end)

Two signals are wired end-to-end as of v1.28.0. Both have existing tooling and cover high-value
surface area (labels touch every repo; security findings carry SLA obligations).

### Signal 1 — Label drift on infrastructure repos

**Trigger**: `bulk-label-audit.sh --category infrastructure` shows repos missing canonical labels.
**Artifact**: PR opened against the target repo with the label diff applied.
**Guard**:
- Autonomy tier `read-only` → skip (no writes allowed)
- Autonomy tier `assist` → skip (human executes, not Claude)
- Autonomy tier `pr-only` → PR opened, human merges
- Autonomy tier `full` → PR opened; auto-merge gated on CI passing
- Rate limit: `--max-closures N` (default 5 per run); dry-run by default

### Signal 2 — Security baseline drift (missing Dependabot alerts, default branch, deploy keys)

**Trigger**: `security-audit.sh all` flags repos with `securityRisk: high|medium` or specific findings.
**Artifact**: Issue filed in target repo with severity label (`priority/critical`, `priority/high`,
`priority/medium`) and remediation instructions.
**Guard**:
- Forks → always skip (we don't own upstream)
- Archived → always skip (no active maintenance)
- Rate limit: `--max-closures N` (default 5 per run); dry-run by default
- Delegation: if the finding is on a non-owned repo, route through `delegate-issue.sh`

## Autonomy Gating

Closure respects `.claude/rules/permission-tiers.md` autonomy levels at all times:

| Autonomy level | PR closures | Issue filing | Notes |
|---|---|---|---|
| `read-only` | Never | Never | Signal logged, no artifact created |
| `assist` | Never | Human-initiated only | Claude advises; human executes |
| `pr-only` | Yes (branch + PR, no merge) | Yes | Canonical default for active repos |
| `full` | Yes (auto-merge if CI green) | Yes | Requires comprehensive CI |

## Blast-Radius Guardrails

All `govern-closure.sh` runs enforce:

1. **Dry-run by default**: `--dry-run` is implied unless `--apply` is passed explicitly.
2. **Max closures per run**: `--max-closures N` (default 5). Prevents runaway loops on a full
   inventory scan. Remaining repos are listed in output with an explanation.
3. **Audit logging**: every closure (PR opened, issue filed, delegation created) is written to
   `data/audit-log.json` via `log_audit` in `tools/lib/gh-helpers.sh`.
4. **Re-run verification**: after closure, re-run the originating audit tool. If the signal still
   appears, the closure is flagged as incomplete in the audit log.

## Planned Closures (not yet wired)

### Template drift

Template drift is detected by `template-drift.sh`. The closure path is to open a PR against
`.github` repos using `sync-github-templates.sh`. Not yet automated because template content
varies more than labels — a human review step before merge is important. Planned for v1.29.0.

### Convention drift (delegation)

When `convention-checker` agent flags a non-owned repo (not in the four-repo core), the closure
path is to file a delegation issue via `delegate-issue.sh --repo <target>`. The target repo's
maintainer ack/wips/reviews per the delegation protocol. Not yet automated end-to-end.

### Convention drift (self-PR)

When the drift is in git-organizer itself (e.g., a convention doc is out of sync with the
reference index), the closure is a self-PR. The `docs/conventions/` files are the source of
truth; the PR updates any stale references. Triggered by `pre-commit-drift.sh` today.
