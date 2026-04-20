# Label Taxonomy

Canonical label set managed by git-organizer. All non-fork repos get these 38 labels synced.

## Core Labels (9)

GitHub's default labels, kept for compatibility:

| Label | Color | Description |
|-------|-------|-------------|
| bug | `#d73a4a` | Something isn't working |
| enhancement | `#a2eeef` | New feature or request |
| documentation | `#0075ca` | Improvements or additions to documentation |
| good first issue | `#7057ff` | Good for newcomers |
| help wanted | `#008672` | Extra attention is needed |
| invalid | `#e4e669` | This doesn't seem right |
| question | `#d876e3` | Further information is requested |
| duplicate | `#cfd3d7` | This issue or pull request already exists |
| wontfix | `#ffffff` | This will not be worked on |

## Type Labels (9)

Classify the nature of work. Every issue/PR should have exactly one:

| Label | Color | Description |
|-------|-------|-------------|
| type/feature | `#0E8A16` | New feature or enhancement |
| type/fix | `#d73a4a` | Bug fix |
| type/refactor | `#1D76DB` | Code refactoring |
| type/docs | `#0075ca` | Documentation |
| type/test | `#BFD4F2` | Tests |
| type/ci | `#FBCA04` | CI/CD changes |
| type/chore | `#D4C5F9` | Maintenance and chores |
| type/security | `#B60205` | Security-related |
| type/config | `#C2E0C6` | Configuration changes |

## Priority Labels (4)

| Label | Color | Description |
|-------|-------|-------------|
| priority/critical | `#B60205` | Must fix immediately |
| priority/high | `#D93F0B` | Important, fix soon |
| priority/medium | `#FBCA04` | Normal priority |
| priority/low | `#0E8A16` | Nice to have |

## Status Labels (5)

Track issue lifecycle:

| Label | Color | Description |
|-------|-------|-------------|
| status/blocked | `#D93F0B` | Blocked by dependency |
| status/in-progress | `#0E8A16` | Actively being worked on |
| status/needs-review | `#1D76DB` | Awaiting review |
| status/needs-testing | `#BFD4F2` | Needs testing |
| status/ready | `#0E8A16` | Ready to be picked up |

## Agent Labels (3)

Control who drives the work:

| Label | Color | Description |
|-------|-------|-------------|
| agent/claude | `#5319E7` | Claude works autonomously |
| agent/human | `#006B75` | Human works independently |
| agent/co-op | `#0E8A16` | Human and Claude collaborate |

## Delegation Labels (6)

Cross-repo delegation state machine. Only one `delegation/*` label active per issue at a time (swap, don't accumulate). The legacy `delegated` label in the status set is retained for backwards compatibility.

| Label | Color | Description |
|-------|-------|-------------|
| delegation/filed | `#D4A574` | Delegation filed — awaiting acknowledgement |
| delegation/ack | `#C5DEF5` | Target repo acknowledged — scheduled |
| delegation/wip | `#0E8A16` | Target actively working |
| delegation/review | `#1D76DB` | Work done — source repo review needed |
| delegation/done | `#0E8A16` | Verified complete by source repo |
| delegation/blocked | `#B60205` | Blocked — needs input from source or target |

State transitions:

```
filed → ack → wip → review → done
  ↓       ↓     ↓       ↓
  └───────┴─────┴───────┘
          blocked
```

See `docs/conventions/delegation-protocol.md` for the full protocol spec.

## Meta Labels (1)

Non-actionable markers about the issue itself:

| Label | Color | Description |
|-------|-------|-------------|
| meta/tracking | `#8B5A00` | Parent/tracking issue — not directly actionable; sub-issues execute the work |

See `docs/conventions/issue-hierarchy.md` for when to use parent/sub-issue structure.

## Category Overrides

Some repo categories get additional labels:

**Infrastructure repos** also get:
- `tool/chezmoi`, `tool/homebrew`, `tool/mise`, `tool/1password`, `tool/fabric`
- `machine/all`, `machine/arechste-mini`, `machine/tutnix`, `machine/corp`, `machine/linux`

## Sync Rules

- Forks: NEVER synced (read-only, we don't own upstream)
- Sync is additive: missing labels added, mismatched colors/descriptions updated
- Extra labels (project-specific): reported but never auto-deleted
- Deletion requires explicit `--delete-extra` flag + human approval

## Commands

```bash
# Diff a repo against canonical labels
./tools/labels/diff-labels.sh <repo-name>

# Sync labels (always dry-run first)
./tools/labels/sync-labels.sh --dry-run <repo-name>
./tools/labels/sync-labels.sh <repo-name>

# Bulk audit
./tools/labels/bulk-label-audit.sh
./tools/labels/bulk-label-audit.sh --category active
```
