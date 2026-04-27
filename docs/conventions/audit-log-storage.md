# Audit Log Storage

`data/audit-log.json` is **local-only and gitignored**. It captures script invocations on the machine that ran them. No other machine reads it; no CI gate depends on it; nothing on `origin/main` consumes its entries. The file exists for human introspection on the local host.

## When to put a file in git (the framework)

A `data/*.json` file belongs on `origin/main` when at least one of these holds:

1. **Distributed consistency** — another machine (a different host, a CI runner, a scheduled routine) reads the file and branches on its contents.
2. **Backup / archive** — the file holds unique data not recoverable elsewhere, and losing it would lose information.

If neither applies, the file is local scratch and should be gitignored. Tracking it taxes every session with a dirty working tree and produces churn PRs that carry no signal.

`data/audit-log.json` fails both criteria:

- **No reader.** No script, skill, CI workflow, routine, or convention enforces anything based on entry contents. The only reads were `git status --porcelain` dirtiness checks (now obsolete).
- **No unique data.** Of the seven highest-volume action types, five (`release-create`, `complete-merge`, `delegate-issue`, `check-delegated-issues`, `prune-merged-branch`) duplicate information GitHub already exposes natively (Releases, PR timelines, target-repo issues, branch-deletion events). Two (`sync-labels`, `security-audit`) carry unique information; those will move to GitHub-native sinks in follow-up work.

By contrast, `data/delegated-issues.json` and `data/delegation-inbox.json` satisfy criterion 1: they are read by `tools/audit/milestone-readiness.sh`, the scheduled `auto-audit-write.yml` workflow, and multiple delegation scripts. Those files stay tracked.

## How `log_audit()` behaves

`tools/lib/gh-helpers.sh:log_audit()` appends one entry per call. On fresh checkouts where `data/audit-log.json` is absent (it is gitignored), the function bootstraps the file with `{"entries":[]}` before appending. No commit, no push — purely local.

`DRY_RUN=true` short-circuits the append so dry-run callers leave the file untouched.

## Lifetime

The file accumulates indefinitely on disk. Pruning is the operator's choice; nothing relies on retention. To start fresh:

```bash
echo '{"entries":[]}' > data/audit-log.json
```

The historical file (entries through PR #388) is preserved at `data/audit-log.archive.json` as a one-time committed snapshot.

## What this means for skills and tools

- Scripts continue to call `log_audit()` as before — no API change. Their writes simply stay local.
- `complete-merge.sh` is now a fully optional wrapper. The merge contract is satisfied by `gh pr merge --delete-branch` alone (see `docs/conventions/merge-completion.md`).
- Sessions never need to "fold an audit entry into the current PR." The file is gitignored; `git status` never surfaces it.
- The SessionEnd dirty-warning hook (dotclaude) is obsolete; removal is tracked in a follow-up delegation.

## Future direction

Two action types carry data that has no GitHub-native equivalent and may move to dedicated GitHub sinks:

| Action | Proposed sink | Status |
|---|---|---|
| `sync-labels` | One long-lived `chore(labels): canonical-label-history` issue per repo; comment per sync run with the diff | **Done** (#390) — `tools/labels/sync-labels.sh` posts a tracking-issue comment on every non-empty sync; local `log_audit()` retained for now |
| `security-audit` | One long-lived `chore(security): audit-history` issue per repo, or rely on the existing finding-issues | Pending follow-up issue |

Once both migrations land, the in-script `log_audit()` calls become vestigial and `tools/lib/gh-helpers.sh:log_audit()` can be removed entirely. Until then, the function stays as a no-cost local appender.

## References

- Issue #388 — the planning issue that motivated this convention
- `docs/conventions/change-verification.md` — verification bar (no longer references audit-log entries)
- `docs/conventions/merge-completion.md` — `complete-merge.sh` is optional, audit logging is local-only
- `tools/lib/gh-helpers.sh:84-110` — `log_audit()` implementation
