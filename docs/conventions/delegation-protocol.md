# Delegation Protocol

Structured cross-repo delegation protocol for machine-parseable feedback between repos.

## Scope

Four core repos participate:

| Repo | Role |
|------|------|
| git-organizer | Publishes labels, schema, reference script. Authoritative source. |
| dotclaude | Distributes protocol to downstream repos via `dc:sync-conventions`. |
| dotfiles | Adopts protocol. Accepts and responds to delegated work. |
| mac-organizer | Adopts protocol. Accepts and responds to delegated work. |

## State Machine

Each delegated issue has exactly one `delegation/*` label at any time. Labels are swapped, not accumulated.

```
filed → ack → wip → review → done
  ↓       ↓     ↓       ↓
  └───────┴─────┴───────┘
          blocked
```

| Label | Applied by | Meaning |
|-------|-----------|---------|
| `delegation/filed` | Source repo (auto) | Issue filed, awaiting acknowledgement |
| `delegation/ack` | Target repo | Acknowledged, will schedule |
| `delegation/wip` | Target repo | Actively working |
| `delegation/review` | Target repo | Work complete, source repo review needed |
| `delegation/done` | Source repo | Verified complete |
| `delegation/blocked` | Either | Blocked, needs input |

### Transition ownership

Each transition has a single owner responsible for applying the label change.

| Transition | Owner | Trigger |
|------------|-------|---------|
| `filed → ack` | Target repo | Auto on session-start detection |
| `ack → wip` | Target repo | Branch created, work begins |
| `wip → review` | Target repo | Work complete, completion report posted |
| `review → done` | **Source repo** | After verification passes |
| `→ blocked` | Either | When blocked, with feedback comment |

The critical gate is `review → done` — the **delegator** (source repo) must verify the work before transitioning. The target repo cannot close the loop unilaterally.

### Transition rules

- **Source repo** can apply: `filed`, `done`, `blocked`
- **Target repo** can apply: `ack`, `wip`, `review`, `blocked`
- When applying a new label, remove the previous `delegation/*` label first
- The legacy `delegated` label is retained alongside `delegation/*` for backwards compatibility

### Auto-close prohibition

Delegated issues MUST NOT use `Fixes #N` or `Closes #N` in PR bodies. Auto-close on merge bypasses the delegator's verification gate.

Instead, the target repo should:
1. Complete work and merge PR normally (no issue-linking keywords)
2. Post a `COMPLETION_REPORT` comment on the delegated issue
3. Transition label: `delegation/wip` → `delegation/review`
4. Post a review-ready notification on the source repo's tracking issue (see below)
5. Wait for the source repo to verify and transition `review → done`

The source repo closes the issue after verification.

### Review-ready notification

When transitioning `wip → review`, the target repo SHOULD post a cross-reference notification on the source repo's tracking issue. This triggers GitHub notifications, eliminating the need for the delegator to manually poll.

**Prerequisite:** The delegated issue body must contain a source reference marker:

```html
<!-- delegation:source repo=SOURCE-REPO issue=N -->
```

This is embedded automatically by `delegate-issue.sh --source-issue N`.

**Notification format:** Post this comment on the source repo's tracking issue:

```html
<!-- delegation:notification from=TARGET-REPO issue=TARGET-REPO#N -->
## [Info]: Delegation ready for review

Work on TARGET-REPO#N is complete and awaiting verification.
Transition: `delegation/wip` → `delegation/review`

See TARGET-REPO#N for the completion report.
```

**Detection:** `check-delegation-feedback.sh` scans for `<!-- delegation:notification` markers alongside standard feedback markers. Notifications appear as `REVIEW-READY` entries in the feedback report.

**Graceful degradation:** If no `<!-- delegation:source -->` marker exists (legacy delegations), the target repo posts only the completion report on the delegated issue. The delegator detects it via `check-delegation-feedback.sh` on the next run.

### Forbidden actions by role

**Target repo MUST NOT:**
- Close the issue (manually or via `Fixes #N` / `Closes #N`)
- Post `[Approval]` or `[Review]` feedback (those types are reserved for the delegator)
- Transition label to `delegation/done`

**Source repo MUST NOT:**
- Transition label to `delegation/ack`, `delegation/wip`, or `delegation/review`
- Post completion reports (that is the target's responsibility)

Violations are detectable by `check-delegation-feedback.sh` (role validation on feedback types).

## Structured Feedback

Comments following this format are machine-parseable by `check-delegation-feedback.sh`:

```html
<!-- delegation:feedback from=REPO-NAME to=REPO-NAME -->
## [Type]: Title

Body text with details.
```

### Feedback types

| Type | When to use | Permitted by |
|------|-------------|--------------|
| `Review` | Reviewing delivered work, acceptance/rejection | Source (delegator) |
| `Question` | Clarification needed before proceeding | Either |
| `Approval` | Approving a proposal or approach | Source (delegator) |
| `Blocker` | Something prevents progress | Either |
| `Info` | Status update, no action required | Either |

### Examples

Source repo reviewing delivered work:
```html
<!-- delegation:feedback from=git-organizer to=dotclaude -->
## [Review]: CLI execution adoption verified

Items 1 and 2 confirmed. Item 3 needs a follow-up for session-start cleanup.
```

Target repo asking a question:
```html
<!-- delegation:feedback from=dotclaude to=git-organizer -->
## [Question]: Label sync scope

Should delegation/* labels be synced to all repos or only infrastructure repos?
```

## Inbox Tracking

Each participating repo maintains `data/delegation-inbox.json` with schema at `data/schemas/delegation-inbox.schema.json`.

### Key fields

| Field | Description |
|-------|-------------|
| `issueRef` | Full qualified reference: `owner/repo#N` |
| `direction` | `inbound` (feedback on issues filed in our repo) or `outbound` (feedback on issues we filed) |
| `commentId` | GitHub comment ID — primary deduplication key |
| `delegationId` | Cross-reference to `delegated-issues.json` entry ID |
| `sourceIssue` | Issue in the delegating project that tracks this delegation |
| `status` | `pending` → `acknowledged` → `resolved` |

### Deduplication

Comments are tracked by `commentId`. The script skips comments already in the inbox.

## Detection Script

`tools/maintenance/check-delegation-feedback.sh`

```bash
# Scan all directions
./tools/maintenance/check-delegation-feedback.sh

# Outbound only (issues we filed in other repos)
./tools/maintenance/check-delegation-feedback.sh --outbound

# Inbound only (issues filed in this repo)
./tools/maintenance/check-delegation-feedback.sh --inbound

# Machine-readable output
./tools/maintenance/check-delegation-feedback.sh --json

# Count only (for hooks)
./tools/maintenance/check-delegation-feedback.sh --quiet

# Write new entries to inbox
./tools/maintenance/check-delegation-feedback.sh --update

# Preview without writing
./tools/maintenance/check-delegation-feedback.sh --update --dry-run
```

### Exit codes

| Code | Meaning |
|------|---------|
| 0 | No new unread feedback |
| 1 | New unread feedback found |
| 2 | Error |

### Session-start integration

Session-start hooks should run `--quiet` to report unread feedback count:

```bash
local count
count=$(./tools/maintenance/check-delegation-feedback.sh --quiet 2>/dev/null || echo "0")
if [[ "${count}" -gt 0 ]]; then
  echo "Delegation feedback: ${count} unread"
fi
```

## Verification Gate

When the target repo transitions to `delegation/review`, the source repo runs verification before transitioning to `delegation/done`.

### Verification methods

Each delegated issue in `delegated-issues.json` may specify a `verifyMethod` and `verifyTarget`.

| Method | What it checks | Target value |
|--------|---------------|-------------|
| `file-exists` | File exists in target repo | Path relative to repo root (e.g., `home/rules/delegation.md`) |
| `file-contains` | File contains a pattern | Path + `verifyPattern` field (regex) |
| `ci-workflow` | CI workflow exists and is valid | Workflow file path (e.g., `.github/workflows/conventions.yml`) |
| `pr-merged` | A linked PR was merged | PR number or `null` (any merged PR on the issue) |
| `manual` | Human-verified | Description of what to check |

### When verification runs

- `check-delegated-issues.sh --update --verify` runs verification on all closed issues with `delegation/review`
- Verification targets the **target repo** (where the work was done), not the source repo
- For `file-exists` and `file-contains`: checks against the target repo's default branch via `gh api`

### On verification failure

1. Post a `[Blocker]` feedback comment explaining what failed
2. Transition label back: `delegation/review` → `delegation/blocked`
3. Target repo addresses the issue and re-transitions to `delegation/review`

### On verification success

1. Post an `[Approval]` feedback comment confirming verification
2. Transition label: `delegation/review` → `delegation/done`
3. Close the issue
4. Update `delegated-issues.json` with `verified: true` and `verifiedAt` timestamp

## Backwards Compatibility

- The flat `delegated` label is always applied alongside `delegation/*` labels
- Scripts that only check for `delegated` continue to work
- `check-delegated-issues.sh` displays `delegation/*` state when available but doesn't require it
- New issues get both labels; old issues with only `delegated` are unaffected

## What Each Repo Provides

### git-organizer (authoritative)

- Label definitions in `data/label-definitions.json`
- Schema at `data/schemas/delegation-inbox.schema.json`
- Reference script: `tools/maintenance/check-delegation-feedback.sh`
- Convention doc (this file)
- `delegate-issue.sh` auto-applies `delegation/filed` + `delegated`

### dotclaude (distributor)

- Propagates labels to downstream repos via `dc:sync-conventions`
- Propagates inbox file + schema to downstream repos
- Extends session-start hooks to detect inbound `delegation/filed`, auto-ack
- Posts structured feedback comments when completing work

### dotfiles / mac-organizer (adopters)

- Accept synced labels, inbox file, and schema from dotclaude
- Session-start hooks detect and report inbound delegated issues
- Post structured feedback when completing delegated work

## Cross-Repo Flow Example

End-to-end walkthrough of a delegation round-trip:

1. **git-organizer** files issue in dotclaude using `delegate-issue.sh --source-issue 42`
   - Labels applied: `delegated`, `delegation/filed`, `type/feature`, `priority/medium`, `agent/claude`
   - Issue body contains `<!-- delegation:source repo=git-organizer issue=42 -->`
   - Tracked in `data/delegated-issues.json`

2. **dotclaude** session starts, detects inbound `delegation/filed` issue
   - Auto-transitions label: `delegation/filed` → `delegation/ack`
   - Reports to user: "1 new delegated issue from git-organizer"

3. **dotclaude** begins work
   - Transitions: `delegation/ack` → `delegation/wip`
   - Posts structured feedback if questions arise

4. **dotclaude** completes work, opens PR
   - Transitions: `delegation/wip` → `delegation/review`
   - Posts completion report comment on the delegated issue
   - Posts review-ready notification on git-organizer#42 (source tracking issue)

5. **git-organizer** receives GitHub notification from the cross-reference
   - `check-delegation-feedback.sh` detects the notification marker
   - Reviews work, runs verification
   - Transitions: `delegation/review` → `delegation/done`
   - Records verification in `delegated-issues.json`

6. Both repos have synchronized state through labels and structured comments
