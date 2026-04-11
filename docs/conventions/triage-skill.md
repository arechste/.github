# Triage Skill Convention

Canonical specification for the `/triage` skill. git-organizer defines the convention; dotclaude distributes it as `dc:triage`.

## Purpose

Triage open issues for any repository: group, prioritize, show milestone readiness, detect blockers, and execute selected work. Bias toward action — present data, ask what to pick up, then do it.

## Frontmatter

```yaml
---
name: dc:triage
description: Triage open issues — group, prioritize, show milestone readiness, execute
tags: [workflow, git, standard]
allowed-tools:
  - Bash(gh issue *)
  - Bash(gh api *)
  - Bash(gh label *)
  - Bash(jq *)
  - Bash(hostname *)
  - Bash(git switch *)
  - Bash(git branch *)
  - Read
  - Grep
  - Glob
---
```

## Steps

### 1. Fetch open issues

```bash
gh issue list --state open --limit 50 \
  --json number,title,labels,milestone,assignees,updatedAt
```

### 2. Group and sort

- Group by milestone (if assigned), then by primary label group (`type/*`, `priority/*`)
- Sort by priority (critical > high > medium > low), then by last updated
- Present a summary table: `#`, Title, Labels, Milestone, Priority, Last Updated

### 3. Milestone readiness

- Fetch ALL open milestones (not just those with open issues):
  ```bash
  gh api repos/{owner}/{repo}/milestones --jq '.[] | {title, open_issues, closed_issues}'
  ```
- Present milestone progress table: Milestone, Open, Closed, %, Status
- For milestones with open issues: flag blockers (`status/blocked`, machine mismatch)
- For milestones at 100%: cross-reference with delegated issues before declaring release-ready

### 4. Check delegated issues (inbound)

```bash
gh issue list --label delegated --state open --json number,title,labels
```

- If present: list prominently with "PRIORITIZE" flag
- Cross-reference with 100% milestones: qualify release-readiness with delegated issue count

### 5. Prompt and execute

- Ask user which issue(s) to work on
- On selection:
  - Claim: `gh issue edit <N> --add-label "status/in-progress"`
  - Branch: create following repo conventions
  - Start implementation immediately

## Convention Decisions

### Autonomy level adaptation

The triage skill adapts its "execute" step based on the repo's autonomy level (from `data/repo-inventory.json`):

| Autonomy | Triage behavior |
|----------|----------------|
| `read-only` | Show issues and milestones only. No claiming, no branching. |
| `assist` | Show issues, suggest commands. User executes. |
| `pr-only` | Full triage: claim, branch, implement, PR. Human merges. |
| `full` | Full triage + may self-merge if CI passes and change is trivial. |

When running outside git-organizer (no inventory), default to `pr-only` behavior.

### Label schema

The skill uses whatever labels exist in the repo. It does not assume git-organizer's canonical labels are present.

- Group by labels that contain `/` (e.g., `type/feature`, `priority/high`)
- Fall back to GitHub defaults (`bug`, `enhancement`) if no `/` labels exist
- Suggest running `/label-sync` if the repo is missing the canonical taxonomy

### Delegated issue scope

Check delegated issues for the **current repo only** (not globally). Cross-repo delegation tracking is git-organizer's concern, not the triage skill's.

For repos that have a `data/delegated-issues.json` (like git-organizer), also check outbound delegated status and flag stale items (>7 days open).

## Rules

- Do NOT enter plan mode — triage is action-oriented
- Do NOT ask for confirmation before labeling or claiming
- Verify labels exist before applying (`gh label list`)
- If an issue is already `status/in-progress`, warn before claiming
- One issue per work session unless user requests batching
- Flag delegated issues as priority — another repo may be blocked
- Suggest `/release` when a milestone is at 100% (qualified by delegated issues)

## Output Format

### Issue Table
Compact markdown table sorted by priority, then last updated.

### Milestone Progress
```
| Milestone | Open | Closed | % | Status |
|-----------|------|--------|---|--------|
```
Milestones at 100%: "Ready for release" with `/release` suggestion (qualified if delegated issues open). Others: show blockers.

### Delegated Issues
Shown after milestones. Prominent "PRIORITIZE" flag.

### Prompt
Ask user which issue(s) to pick up.
