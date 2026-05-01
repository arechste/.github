# Ledger Issues

A **ledger issue** is a long-lived GitHub issue that exists as a write target for tooling. It is intentionally kept open and never closes through normal release cadence. Tools post comments to it (one per non-empty event) so the issue's comment thread becomes an audit log.

## Examples

- `chore(labels): canonical-label-history` (#394) — `tools/labels/sync-labels.sh` posts the diff applied to a repo's labels on each non-empty run.

## Properties

| Property | Value |
|----------|-------|
| Closes | Never (re-opened by tooling on next non-empty write) |
| Milestone | None — never assigned to a release milestone |
| Bucket | Excluded from `Backlog` (Backlog implies "deferred work") |
| Pickup | Skipped by `/dc:triage` and other pickup workflows |
| Label | `meta/ledger` (required) |
| Type label | Whatever fits the domain (often `type/chore`) |
| Priority | Usually `priority/low` (informational) |

## Why a separate convention

Ledger issues don't fit either of the two normal buckets:

- **Backlog** is for *deferred but actionable* work — a ledger never becomes work.
- **Version milestones** are *release-shaped* — a ledger never ships.

Without distinction, ledger issues get surfaced by triage as "ready" candidates and create noise. The `meta/ledger` label disambiguates them at the source.

## Convention

A ledger issue MUST:

1. Carry the `meta/ledger` label.
2. Not be assigned to any milestone (Backlog included).
3. State its purpose explicitly in the body — what writes to it, when, and a link to the writing tool.
4. Be safe to close manually — the writing tool MUST reopen it on the next non-empty write.

A ledger issue SHOULD:

- Use `priority/low` (or omit priority — informational, not work).
- Use `agent/co-op` (or omit agent — no agent owns it; the tool maintains it).

## Tooling integration

- `/dc:triage` excludes `meta/ledger` issues from the issue table and "pick up" prompt. They appear in a separate "Ledgers" section if any exist.
- `tools/audit/build-forge-index.sh` and similar inventory scripts treat them as metadata, not work.
- `tools/labels/sync-labels.sh` is the canonical example writer — see #394 for the format.

## Creating a new ledger issue

1. Open the issue with title `chore(<scope>): <ledger-name>` (e.g., `chore(audit): security-scan-history`).
2. Apply labels: `type/chore`, `priority/low`, `meta/ledger`.
3. Do **not** assign a milestone.
4. Body should state: what tool writes to it, what trigger (cron, manual, post-merge, etc.), and a one-line note that the issue is intentionally permanent.
5. Reference the issue number from the writing tool so writes route correctly.

## Cross-references

- `docs/conventions/label-taxonomy.md` § Meta Labels — `meta/ledger` definition
- `docs/conventions/audit-log-storage.md` — broader audit-log strategy (in-repo vs. issue-based)
- `docs/conventions/issue-hierarchy.md` — sibling concept: tracking issues (parent of work) vs. ledger issues (write target only)
