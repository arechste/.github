# Project Templates

Governed templates for GitHub Projects v2. Two project types serve different scopes.

> **Cross-repo overview note** — Program-level tracking (formerly Project #15 "Core Infrastructure") is **retired**. Cross-repo visibility is now produced by `tools/audit/cross-repo-status.sh` per [`oversight-strategy.md`](oversight-strategy.md). The Global template (Project #12) remains for triage workflows.

## Project Types

| Type | Scope | When to use |
|------|-------|-------------|
| **Global** | All repos (or a broad set) | Portfolio-level triage, status tracking, bug oversight |
| **Repo** | Single repo | Only when a repo needs its own board (most should use Global) |

## Shared Field Schema

All project types use the same base fields for consistency.

### Status (Single Select)

| Option | Color | Description |
|--------|-------|-------------|
| Backlog | Gray | Filed but not triaged or not ready |
| Ready | Blue | Requirements clear, can be picked up |
| In Progress | Yellow | Actively being worked on (branch exists) |
| In Review | Orange | PR open, awaiting review |
| Done | Green | PR merged, issue closed |

### Priority (Single Select)

| Option | Color | Description |
|--------|-------|-------------|
| Critical | Red | Immediate action required |
| High | Orange | Important, schedule soon |
| Medium | Yellow | Normal priority |
| Low | Blue | Nice to have |

### Size (Single Select)

| Option | Effort |
|--------|--------|
| XS | < 30 min |
| S | 30 min - 2 hours |
| M | 2-4 hours |
| L | 4-8 hours |
| XL | 1-2 days |

### Built-in Fields (always present)

Title, Assignees, Labels, Repository, Milestone, Linked PRs, Parent issue, Sub-issues progress.

## Automation (all types)

Enable these built-in project workflows:

| Trigger | Action |
|---------|--------|
| Item added | Status → Backlog |
| Item reopened | Status → In Progress |
| Item closed | Status → Done |
| PR merged | Status → Done |

Label-to-status sync (e.g., `status/ready` → Ready column) is not natively supported by GitHub Projects. Accept the gap — manual moves during triage handle transitions. Revisit with GitHub Actions if drift becomes a problem.

## Global Template

**Purpose:** Portfolio-level visibility across all repos.

**Additional fields:** None beyond shared schema.

**Auto-add:** `is:issue is:open` across all target repos.

**Views:**

| View | Layout | Filter / Group | Purpose |
|------|--------|---------------|---------|
| Triage | Table | status:Backlog, group by Repository, sort oldest | Process new items |
| Active Work | Board | status:Ready/In Progress/In Review, columns=Status | Kanban of in-flight work |
| By Repository | Table | exclude Done, group by Repository, sort by Priority | Repo-level planning |
| Bugs | Table | label:type/fix or bug, exclude Done | Bug tracker |
| Blocked | Table | label:status/blocked, sort by Priority | Surface stuck items |
| Recently Closed | Table | status:Done, sort most recent | Throughput review |

**Live instance:** "arechste: All Issues" (project #12)

## Repo Template

**Purpose:** Per-repo board for repos needing their own project.

**Additional fields:** None beyond shared schema.

**Auto-add:** Single repo, issues only.

**Views:**

| View | Layout | Filter / Group | Purpose |
|------|--------|---------------|---------|
| Board | Board | columns=Status | Kanban |
| Backlog | Table | status:Backlog, sort by Priority | Triage queue |

**When to use:** Most repos should NOT need their own project. Use Global for visibility and `tools/audit/cross-repo-status.sh` for cross-repo coordination. Only create a Repo project when a repo has enough issues to justify dedicated views (>20 active items).

**Used by:** `/repo-setup` skill during scaffolding (optional, not default).

## Repo-to-Project Mapping

### Decision Tree

```
Is the repo a fork?
  └─ Yes → No project membership (forks are read-only)
  └─ No →
      Is the repo dormant or profile?
        └─ Yes → No project membership (no active issues)
        └─ No → Global (#12) only
```

Cross-repo coordination (formerly Program-level tracking) is now provided by `tools/audit/cross-repo-status.sh`, not by a project board. See [`oversight-strategy.md`](oversight-strategy.md).

### Category-to-Project Matrix

| Category | Global (#12) | Notes |
|----------|:---:|-------|
| infrastructure | Yes | Core repos: dotfiles, dotclaude, git-organizer, mac-organizer |
| active | Yes | Cross-repo coordination via `cross-repo-status.sh` |
| learning | Yes | Low-volume, mostly for visibility |
| dormant | — | No active issues; re-add if reactivated |
| fork | — | Read-only; upstream owns the project board |
| profile | — | .github, .pai — metadata repos, rarely have issues |

### Onboarding Checklist (New Repo)

When `/repo-setup` bootstraps a new repo:

1. Labels synced from `data/label-definitions.json`
2. Milestones created (Backlog + first version milestone)
3. Repo linked to Global project (#12) via `gh project link`
4. Reminder printed for UI-only steps: configure auto-add rule (if plan allows)

### Agent identities

Agent identities (Auditor, Builder, Ops, Curator) are tracked via commit-trailer
(see [`commit-format.md`](commit-format.md)) and `agent/*` issue labels (see
[`../../.claude/rules/agent-modes.md`](../../.claude/rules/agent-modes.md)). The
dedicated **Agent** project field used by the retired Program template is no
longer needed; identity lives in commit history and label state.

### ntnxlab-ch Expansion

When adding ntnxlab-ch repos:

- Cross-org sub-issues are supported (since Sep 2025)
- Auto-add workflows are per-project and can't span orgs — use `actions/add-to-project` GitHub Action
- Cross-org overview goes through `tools/audit/cross-repo-status.sh` (forge-portable) rather than a parallel org-level Project board
- Global project (#12) stays scoped to arechste org
- Shared conventions (labels, templates) deploy via `/sync-conventions` skill

## Creating a Project from Template

```bash
# 1. Create the project
gh project create --owner OWNER --title "Project Title"

# 2. Update Status field (replace default Todo/In Progress/Done)
# Use GraphQL mutation: updateProjectV2Field with singleSelectOptions

# 3. Add custom fields
gh project field-create NUMBER --owner OWNER --name "Priority" \
  --data-type "SINGLE_SELECT" --single-select-options "Critical,High,Medium,Low"
gh project field-create NUMBER --owner OWNER --name "Size" \
  --data-type "SINGLE_SELECT" --single-select-options "XS,S,M,L,XL"

# 4. Link repos
gh project link NUMBER --owner OWNER --repo "OWNER/REPO"

# 5. Bulk import existing open issues
gh issue list --repo "OWNER/REPO" --state open --json url --jq '.[].url' | \
  while read -r url; do
    gh project item-add NUMBER --owner OWNER --url "$url"
    sleep 0.5
  done

# 6. Configure views and automations in the GitHub UI
# (View creation is not available via API)
```

## Limitations

- **Views** cannot be created, deleted, or configured via the GitHub API or CLI. Must be set up manually in the GitHub UI.
- **Auto-add rules** must be configured in the GitHub UI under Project Settings > Workflows.
- **Auto-add limits by plan**: Free=1 rule, Pro=5, Team=5, Enterprise=20. Only catches new/updated items — not retroactive.
- **Label-to-status sync** is not supported natively. Built-in automations only handle: item added, reopened, closed, PR merged.
- **Field option colors** require GraphQL mutations to set (not supported by `gh project field-create`).
- **Item limit**: 50,000 per project (GA April 2025).
- **Issue dependencies**: blocked-by/blocking GA (Aug 2025), up to 50 per issue, cross-repo. No "relates to" or graph visualization yet.
- **Sub-issues**: up to 100 per parent, 8 nesting levels. Inherit Project and Milestone from parent.
- **No full-text search** across project item bodies from the project view — use GitHub issue search separately.

## UI Operations Guide

All view and automation configuration is **UI-only** — the GitHub API does not support view management. This guide documents the manual steps.

### Creating a View

1. Open the project on GitHub (github.com/orgs/OWNER/projects/NUMBER or github.com/users/OWNER/projects/NUMBER)
2. Click **+ New view** (top tab bar, right of existing views)
3. Choose layout: **Table**, **Board**, or **Roadmap**
4. Name the view by clicking the default name and typing
5. Configure using the toolbar controls (see below)
6. Changes auto-save

### Configuring Table Views

| Control | Location | Action |
|---------|----------|--------|
| **Add column** | Click **+** at the right edge of the header row | Select a field to add as a column |
| **Remove column** | Right-click column header → Hide field | Hides field from this view only |
| **Filter** | Filter icon (funnel) in toolbar | Syntax: `field:value`, `-field:value`, `no:label` |
| **Group by** | Group icon (stacked bars) in toolbar | Select a field — items collapse into groups |
| **Sort** | Sort icon (arrows) in toolbar | Select field + direction (asc/desc) |
| **Save** | Auto-saves on change | — |

### Configuring Board Views

| Control | Location | Action |
|---------|----------|--------|
| **Column field** | Board defaults to Status | Change via View menu → Column field |
| **Column limit** | Click column header → Set limit | Max items per column (WIP discipline) |
| **Slice by** | Slice icon in toolbar | Secondary grouping dimension (e.g., Repository) |
| **Filter** | Filter icon in toolbar | Same syntax as Table views |

### Configuring Roadmap Views

| Control | Location | Action |
|---------|----------|--------|
| **Date field** | View menu → Date field | Select which date/iteration field drives positioning |
| **Zoom level** | Zoom control in toolbar | Month, Quarter, or Year |
| **Group by** | Group icon in toolbar | Group timeline rows by a field |
| **Sort** | Sort icon in toolbar | Order groups or items |

### Setting Up Auto-Add Rules

1. Open project → **Settings** (gear icon, top right) → **Workflows** (left sidebar)
2. Click **Auto-add to project**
3. Configure filter: repository, `is:issue`, `is:open`, label filters
4. Toggle **Enabled**
5. Repeat for additional rules (up to plan limit: Free=1, Pro=5)

**Workaround for Free plan**: Use the `actions/add-to-project` GitHub Action in repo workflows to auto-add items beyond the single rule limit.

### Enabling Built-in Automations

1. Open project → **Settings** → **Workflows** (left sidebar)
2. Enable each workflow:
   - **Item added to project** → Set status to **Backlog**
   - **Item reopened** → Set status to **In Progress**
   - **Item closed** → Set status to **Done**
   - **Pull request merged** → Set status to **Done**
3. Only "Item closed" and "PR merged" are enabled by default — enable the others manually

### Field Visibility Per View

Each view has independent field visibility. To add a field to a specific view:

1. Navigate to the view
2. For Table: click **+** at the right of the header row, select field
3. For Board: fields show on item cards — click item to see all fields, or use **View menu → Fields** to toggle card fields
4. Changes apply to the current view only

### Chrome MCP Usage Protocol

When using Chrome MCP to automate UI operations:

1. Ask user permission before starting
2. Confirm which machine's browser to use
3. Take screenshots before/after for verification
4. Clean up any tabs created during the session
