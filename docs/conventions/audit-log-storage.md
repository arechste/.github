# Audit Log Storage

> **Status: removed (April 2026 — issues #391, #392).**

`data/audit-log.json` no longer exists. The `log_audit()` helper and the `AUDIT_LOG` constant have been removed from `tools/lib/gh-helpers.sh`. The earlier `data/audit-log.archive.json` snapshot has been deleted as well — the archived entries duplicated information that GitHub already exposes natively (Releases, PR timelines, target-repo issues, branch-deletion events).

## Where the per-action evidence lives now

| Concern | Replaced by |
|---|---|
| Audit verification trail (`securityAudit`, `maturityAudit`) | `data/repo-inventory.json` already carries `securityAuditedAt`, `maturityAuditedAt`, `labelsAuditedAt`, etc. |
| Operational receipts for write actions (label sync, releases, delegations) | The artifact itself — the comment on the per-repo `chore(labels): canonical-label-history` tracking issue (`workflows/label-sync.md`), the GitHub release, the delegated issue. |
| Read-side bookkeeping (`check-delegated-issues`) | Dropped — it was only ever metadata about *when we looked*. |

## Why this stub still exists

Per-repo tracking issues created by earlier runs of `tools/labels/sync-labels.sh` link here from their issue body. Keeping this file as a tombstone preserves those links. New runs of `sync-labels.sh` link to `workflows/label-sync.md` instead.

## References

- Issue #388 — planning issue (audit-log strategy decision)
- Issue #391 — decision: drop the security-audit sink (option 2)
- Issue #392 — cleanup PR that removed `log_audit()` and the file
- `docs/conventions/merge-completion.md` — `complete-merge.sh` is now purely a squash-merge verification wrapper, no audit logging
- `workflows/label-sync.md` — current home of the label-sync workflow
