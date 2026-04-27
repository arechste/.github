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

## Two Variants

Cross-repo issues fall into two shapes. Pick the variant up front — they differ in labels, state machine, and who closes.

| Variant | Intent | Labels | State machine | Closes when |
|---------|--------|--------|---------------|-------------|
| **Do-this-work** (default) | "Please do X in your repo." One repo owns the outcome; the other verifies. | `delegated` + `delegation/*` (full state machine) | `filed → ack → wip → review → done` | Source repo verifies and transitions `review → done` |
| **Bilateral-spec** | "Let's align on X that affects both our repos." No single owner — both sides co-decide. | `delegated` only (no `delegation/*`) | None | Primary issue closes; mirror closes in lockstep |

### Do-this-work variant

The rest of this document (state machine, verification gate, feedback types, inbox tracking) describes the do-this-work variant. Filed via `tools/maintenance/delegate-issue.sh`.

### Bilateral-spec variant

For cross-repo spec discussions where two repos need to agree on an interface, convention, or boundary that neither repo owns alone (e.g., dotfiles ↔ mac-organizer responsibility split).

**Pattern:**

1. **Primary issue** in one repo carries the full proposal body, questions, and proposed defaults.
2. **Mirror issue** in each counterpart repo carries a short pointer: "discussion lives in `owner/repo#N` — respond there."
3. Both issues get the `delegated` label; no `delegation/*` state is applied.
4. Discussion stays on the primary. Maintainers of each side ack, push back, or amend inline.
5. When the primary closes, mirrors close in lockstep.

**Rules:**

- `delegation/*` labels MUST NOT be applied — there is no `wip`/`review` gate because no work product is being delivered to a delegator.
- Mirrors MUST link to the primary with a clearly-labeled "Where to respond" section.
- Mirror bodies SHOULD stay small — duplicated content causes divergence.
- If a bilateral discussion produces work that one side must then execute, file a **separate** do-this-work issue for that execution. Don't retrofit state-machine labels onto the spec issue.
- If one repo is the **convention authority** for one of the questions (git-organizer for labels, CI, delegation itself), file a sibling do-this-work issue in git-organizer alongside the bilateral. That issue uses the full state machine and closes once the ruling is documented. Example: dotfiles#583 and mac-organizer#218 are bilateral; git-organizer#343 is the paired authority issue.

**When to pick which:**

- If one repo can satisfy the acceptance criteria alone → **do-this-work**.
- If acceptance requires agreement across repos before anyone acts → **bilateral-spec**.
- Hybrid (spec + execution) → one bilateral + one or more do-this-work siblings.

**Rationale:** ruled by #343 on 2026-04-21. Before formalization, mirror issues drifted toward the state machine ("should the mirror be `delegation/wip` while discussion is open?"), which never fit — neither side owns the work product. Separating the two variants keeps the state machine semantically clean (every transition corresponds to work delivered) and gives bilateral spec discussions their own, simpler protocol.

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

## Alt-Paths

The happy path (`filed → ack → wip → review → done`) covers most delegations. These are not protocol bugs — they are legitimate, rare close shapes where the full state machine would be misleading. Each alt-path has a defined owner, label sequence, and completion-report format so delegator and target agree on what happened without free-form notes.

### Subsumed

The work is folded into a different or larger issue; no separate PR exists for this delegation.

**When:** The target repo discovers the delegated change is already being made as part of a broader effort, and splitting it out would create churn or duplication.

**Who acts:**
1. **Target repo** transitions `→ delegation/review` and posts a completion report:
   - `artifacts`: the absorbing issue/PR reference (e.g., `owner/repo#N`)
   - `validation`: confirmation the delegated scope is covered by the absorbing work
   - `deviations`: `subsumed by <owner/repo#N> — no standalone PR`
2. **Source repo** verifies the absorbing issue/PR covers the acceptance criteria, then transitions `delegation/review → done` and closes.

**Note:** The source repo closes the original delegated issue — never the target.

### Not-planned / Closed-not-planned

The underlying premise changed — decision reversed, scope dropped, dependency replaced — so the work will not be done.

**When:** The target repo or delegator determines the delegation is no longer valid (e.g., the feature it supported was removed, an ADR was superseded, the approach was abandoned).

**Who acts:**
1. **Either repo** can initiate: post a `[Info]` feedback comment explaining the premise change. Tag the other repo for acknowledgement.
2. **Source repo** transitions `→ delegation/done` and closes the issue as "not planned" via GitHub's close-as-not-planned option.
3. Completion report is posted by whichever side is closing: `deviations: closed-not-planned — <one-line reason>`.

**Note:** `delegation/review` is skipped — there is no work product to verify. The source repo closes directly after both sides acknowledge.

### Closed-without-review

The target repo closed the issue (e.g., GitHub auto-close via a linked PR, or a direct close) without going through `delegation/review`. The delegator must backfill verification.

**When:** Typically a protocol miss — `Fixes #N` was used in a PR, or the issue was manually closed before posting the completion report.

**Who detects:** `check-delegation-feedback.sh` surfaces closed issues still at `delegation/wip` or `delegation/ack` (no `review` transition on record).

**Backfill procedure:**
1. **Source repo** checks whether the work was actually completed (review the linked PR, commit, or target branch).
2. If verified complete: transition the closed issue's label to `delegation/done`, post a `[Review]` feedback comment confirming backfill, and update `delegated-issues.json` with `deviations: closed-without-review — verified via <PR/commit ref>`.
3. If NOT verified: reopen the issue, post a `[Blocker]` comment requesting a completion report, transition back to `delegation/wip`.

**Prevention:** Always post a completion report and transition to `delegation/review` before any close path. Never use `Fixes #N` / `Closes #N` in PR bodies for delegated issues.

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
- Canonical schemas: `data/schemas/delegated-issues.schema.json` and `data/schemas/delegation-inbox.schema.json`
- Reference scripts: `tools/maintenance/check-delegation-feedback.sh`, `check-delegated-issues.sh`, `delegate-issue.sh`
- Convention doc (this file)
- `delegate-issue.sh` auto-applies `delegation/filed` + `delegated`

### dotclaude (distributor / catalyst)

- Carries the canonical-schema mirror at `data/schemas/{delegated-issues,delegation-inbox}.schema.json` (synced via `dc:sync-conventions schemas`)
- Cross-machine availability — dotclaude lives on every fleet machine, so its mirror is the always-reachable copy
- Propagates labels and inbox/outbox-file conventions to downstream repos
- Extends session-start hooks to detect inbound `delegation/filed`, auto-ack
- Posts structured feedback comments when completing work
- `/dc:delegate` skill writes tracker entries conforming to the canonical schema

### dotfiles / mac-organizer (adopters)

- Maintain their own `data/delegated-issues.json` and `data/delegation-inbox.json` tracker files (data, not schema)
- Tracker `$schema` fields point at dotclaude's mirror (raw URL); validators resolve locally to `~/dotclaude/data/schemas/`
- Do NOT carry their own `data/schemas/` directory
- Session-start hooks detect and report inbound delegated issues
- Post structured feedback when completing delegated work

## Schema Authority

This section codifies the cross-repo schema topology established 2026-04-26 (parent issue: `chore(delegation): standardize tracker schemas across fleet`).

### Canonical files (git-organizer)

| File | Purpose |
|---|---|
| `data/schemas/delegated-issues.schema.json` | Outbound delegation tracker — issues this repo filed in others |
| `data/schemas/delegation-inbox.schema.json` | Inbound feedback tracker — structured comments on delegated issues |

git-organizer is the only repo authorized to **edit** these schemas. All changes flow downstream from here.

### Distribution (dotclaude)

dotclaude carries a synced mirror at the same paths, with a top-level `$comment` provenance marker:

```
"$comment": "Source: git-organizer/data/schemas/<file>. Mirrored via dc:sync-conventions schemas. Do not edit here — propose changes upstream."
```

Run `dc:sync-conventions schemas` in dotclaude to refresh the mirror. The skill diffs against git-organizer's source and prompts before writing.

### Tracker shape (every repo)

Tracker files (the data, not the schema) live in each repo's `data/` directory. Both files MUST use a wrapper-object root:

| File | Canonical shape |
|---|---|
| `data/delegated-issues.json` | `{ "$schema": "<url>", "issues": [...] }` |
| `data/delegation-inbox.json` | `{ "$schema": "<url>", "version": "1.0.0", "lastChecked": null, "entries": [...] }` |

Bare-array form (`[ ... ]`) is INVALID. The `/dc:delegate` skill initializes a missing tracker from the wrapper template before appending. Tools that read these files (`check-delegated-issues.sh`, `check-delegation-feedback.sh`) require the wrapper.

### Adopter `$schema` reference

Adopter repos (dotfiles, mac-organizer) do NOT carry a local `data/schemas/` directory. Their tracker files reference dotclaude's mirror via raw URL:

```
"$schema": "https://raw.githubusercontent.com/arechste/dotclaude/main/data/schemas/delegated-issues.schema.json"
```

Validators rarely fetch this URL (all repos private — auth required), but the URL identifies the contract. Tools resolve the schema locally via `~/dotclaude/data/schemas/...` since dotclaude is on every fleet machine.

### `delegation-outbox.json` is DEPRECATED

The legacy `data/delegation-outbox.json` file (different shape from `delegated-issues.json`) is deprecated. The `/dc:delegate` skill explicitly does not write to it. Per the parent tracking issue:

1. Each repo migrates outbox entries into `delegated-issues.json` (mapping below).
2. Each repo deletes `delegation-outbox.json`.
3. Removal is gated on convergence — no unilateral cleanup.

Outbox-to-issues field mapping:

| outbox field | issues field | notes |
|---|---|---|
| (derive) | `id` | `<repo>-<title-slug>-<issueNumber>` |
| `targetRepo` short name | `repo` | strip `arechste/` |
| extract `N` from `issueRef` | `issueNumber` | `arechste/dotclaude#383` → 383 |
| `title` | `title` | |
| `done` → `closed`; else → `open` | `status` | |
| `done` → `done`; else → `filed` | `delegationState` | |
| `filedAt` | `filedAt` | |
| `closedAt` or null | `closedAt` | |
| null | `lastChecked`, `lastNudgedAt`, `verified`, `verifiedAt` | |
| `manual` | `verifyMethod` | |
| null | `verifyTarget`, `verifyPattern`, `completionReport` | |

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
