# Issue Hierarchy

When to use GitHub sub-issues vs flat issues, and how parent/child issues interact with milestones, labels, branches, and PRs.

GitHub sub-issues are GA since April 2025. Free tier supported. See `data/references/source_github_sub_issues.json` for API details.

## Decision Rule

Use **sub-issues** when ANY of:

- Work spans **3+ PRs**
- Work spans **multiple sessions**
- Work mixes **local execution + cross-repo delegation**
- Progress visibility across an initiative matters (stakeholder tracking, release planning)

Stay **flat** otherwise:

- Single-PR work
- Single-session tasks
- Simple bug fixes
- Atomic refactors

When uncertain, stay flat. Promoting a flat issue to a parent mid-flight is cheap; unwinding an over-structured hierarchy is not.

## Roles

| Role | Purpose | Actionable? |
|------|---------|-------------|
| **Parent issue** | Tracks an initiative, rolls up progress | No — parents are not directly worked |
| **Sub-issue** | Executable unit of work | Yes — one branch, one PR |

A parent issue describes the goal and links sub-issues. It has no branch and no PR of its own. All code changes live on sub-issue branches.

## 1:1:1 Invariant

The 1:1:1 mapping (issue → branch → PR) applies **at the sub-issue level**, not the parent level:

| Layer | Parent | Sub-issue |
|-------|--------|-----------|
| Issue | Yes | Yes |
| Branch | No | One |
| PR | No | One |

A parent issue with 5 sub-issues produces 5 branches and 5 PRs, not 1 aggregate PR.

## Milestones

- **Parent gets the milestone.** Children inherit implicitly.
- Do NOT assign the same milestone to both parent and child — it double-counts in milestone progress reports.
- Milestone readiness rolls up child completion to the parent (see `docs/conventions/semver.md`).

## Labels

Parent issues get a **tracking label** to distinguish them from actionable work:

| Label | Applied to | Meaning |
|-------|-----------|---------|
| `meta/tracking` | Parent issues | Not directly actionable — sub-issues execute the work |

Sub-issues use standard labels (`type/*`, `priority/*`, `agent/*`, `status/*`) — they behave like flat issues.

Triage tooling filters `meta/tracking` out of the "ready to work" view so parents never appear as pickable tasks.

## Close Policy

- **Sub-issues** close normally when their PR merges (`Fixes #N` auto-close works).
- **Parent issues** close **manually** after verifying all sub-issues are closed.
- Parents are NEVER auto-closed. Do not use `Fixes #PARENT` in any PR body — parents span multiple PRs.

Closing a parent is a deliberate act:

1. Verify all sub-issues closed
2. Post a summary comment (what shipped, what deferred)
3. Manually close

## Delegation

Delegated issues stay **flat in the target repo**. A parent issue in git-organizer may have a delegated sub-issue that is itself flat in, say, `dotclaude`.

- The delegation verification gate (`delegation/review` → `delegation/done`) is unchanged.
- The parent in the source repo tracks the delegation as a sub-issue; the child issue in the target repo is a standalone flat issue.
- `parentIssue` field in `delegated-issues.json` records the source-repo parent (see issue #241).

## Promotion Pattern

### At creation (planned multi-part work)

Recommended when the scope is known up front:

1. File the parent with the initiative description, `meta/tracking` label, milestone
2. File each sub-issue and link to parent via `gh api` (see `tools/lib/gh-helpers.sh` — planned in #237)
3. Assign milestone to parent only

### Mid-flight (scope grew)

When a flat issue turns out to be multi-part:

1. Promote the flat issue to a parent: add `meta/tracking` label, edit description to summarize scope
2. File new sub-issues for remaining work
3. Link existing PRs (already merged or in flight) to sub-issues, not the parent
4. Keep the original issue number as the parent — don't renumber

Do not retroactively promote issues whose work is already complete. Flat-to-parent promotion only makes sense when there is still work to scope.

## Anti-Patterns

- **Parent with one sub-issue** — collapse to a flat issue
- **Deep nesting** — sub-issues of sub-issues. Keep hierarchy 2 levels max (parent → child)
- **Parent with its own branch or PR** — split the parent's scope into a dedicated sub-issue
- **`Fixes #PARENT` in a PR** — auto-closes the parent prematurely
- **Same milestone on parent and children** — double-counts progress
- **Using sub-issues for unrelated work** — hierarchy implies shared goal, not batching convenience

## Tooling Alignment

Skills and scripts that need sub-issue awareness (tracked separately):

| Tool | Change | Issue | Status |
|------|--------|-------|--------|
| `gh-helpers.sh` | Add sub-issue REST helpers | #237 | shipped |
| `triage` | Group sub-issues under parents | #238 | shipped |
| `batch` | Execute sub-issues of a parent via `--parent` | #239 | shipped |
| `ship` / `milestone-readiness` | Roll up parent completion | #240 | shipped |
| `delegate-issue.sh` | Track `parentIssue` field | #241 | open |
