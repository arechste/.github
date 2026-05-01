# Cross-Repo Oversight Strategy

## Decision

**git-organizer keeps code-related oversight inside self-hostable forges.** No external SaaS tracker (Linear, ClickUp, Notion, etc.) is part of the canonical surface for code work. Cross-repo visibility is provided by a portable DIY tool (`tools/audit/cross-repo-status.sh`) that aggregates per-repo state via the forge's REST API.

External trackers may stay personal-only (todos, household, business context outside code). That's out of scope here and does not need a convention.

## Status

**Active** — supersedes the prior reliance on GitHub Projects v2 (Project #15) for cross-repo overview.

## Context

The 2026-04-25 organize-self audit (`docs/audit/2026-04-25-organize-self.md`) flagged Project #15 + `tools/maintenance/project-setup.sh` as deprecation candidates because:

- Claude can't query Projects v2 with the current token (lacks `read:project` scope)
- Views require manual UI work — no API for view layout
- Cross-repo "Blocked By" tracking exists but isn't actually consumed
- Field option colors require GraphQL mutations not exposed by `gh project`

These are *structural* limitations, not pricing-driven. Paying for a higher GitHub tier doesn't fix them.

Issue #403 then asked two strategic questions:

1. **DIY portability** — if we DIY, what data model survives a forge migration to GitLab or Forgejo?
2. **Build vs. buy** — should we adopt a privacy-respecting external tracker for some scope?

## Alternatives considered

### (a) External tracker for personal/business only; GitHub canonical for code — **selected**

External SaaS for non-code todos remains a personal choice and is out of scope here. For code, GitHub stays canonical.

- Pros: clean separation; respects `project_forge_vision` (self-hosted goal) and `project_local_first_workflow`; no third-party privacy exposure for code/issue content; no per-user fees
- Cons: no native single-pane-of-glass for code + life

### (b) External tracker for cross-repo overview; per-repo work in GitHub

Keep code in GitHub but mirror summaries to Linear/ClickUp.

- Pros: rich UI, mature dashboards
- Cons: two systems to keep in sync; privacy exposure of issue titles/labels; locks the cross-repo view to one vendor; works against the forge-portability goal; recurring per-user cost

### (c) Reject external; DIY everything

DIY both the per-repo pipeline (already done) and the cross-repo overview.

- Pros: portability; privacy; no recurring cost
- Cons: more code to maintain; output less polished than a SaaS dashboard

The selected path is **(a) for non-code + DIY for code**, which combines the benefits of (a) and (c) without conflict.

## Implementation outline

### 1. Retire GitHub Projects v2 dependency

- Deprecate `tools/maintenance/project-setup.sh` (per audit recommendation)
- Mark Project #15 as archived; do not migrate items
- Update `docs/conventions/project-templates.md` to reference Project #12 only (Global) — Program-level tracking moves to the new tool

### 2. Build `tools/audit/cross-repo-status.sh`

Portable cross-repo aggregator. Output is forge-agnostic JSON; renderers add Markdown / terminal output.

**Data model**

```json
{
  "generatedAt": "ISO-8601",
  "forge": "github",
  "repos": [
    {
      "name": "string",
      "url": "string",
      "openIssues": 0,
      "openPRs": 0,
      "openDelegations": [{ "number": 0, "title": "", "label": "delegation/wip" }],
      "milestones": [{ "title": "", "openCount": 0, "closedCount": 0, "percent": 0 }],
      "lastReleaseTag": "string|null",
      "ciStatus": "passing|failing|disabled|unknown"
    }
  ],
  "crossRepoBlockers": [
    { "issue": "owner/repo#N", "blockedBy": ["owner/repo#M"] }
  ]
}
```

**API surface**

- REST + `jq` for the default GitHub case (`gh api`, `--cache 5m` per `.claude/rules/gh-api-usage.md`)
- GraphQL only when needed for forge-specific features (kept thin so adapters stay simple)
- One adapter per forge: `lib/adapters/github.sh`, `lib/adapters/gitlab.sh`, `lib/adapters/forgejo.sh`

**Output formats**

```
./tools/audit/cross-repo-status.sh                  # JSON to stdout
./tools/audit/cross-repo-status.sh --md             # Markdown for issue/README posting
./tools/audit/cross-repo-status.sh --pretty         # Terminal-pretty for ad-hoc check
./tools/audit/cross-repo-status.sh --forge gitlab   # Use the GitLab adapter
```

**Filters**

- `--category active` — limit to a category from `repo-inventory.json`
- `--maturity ≥mvp` — limit by maturity score
- `--include-delegations` — include delegation state machine view

### 3. Sacrificed for portability

- Projects v2 custom fields, custom views, and automations
- The audit already flagged these as broken in our context — losing them is the goal

### 4. Forge migration pathway

- GitHub → GitLab: REST issue API offers full parity at the Free tier (verified via `docs.gitlab.com/api/issues/`)
- GitHub → Forgejo: Swagger-introspected endpoints; same data model, new adapter only

## Triggers to revisit

This decision should be reviewed if any of the following hold:

- Linear or ClickUp adds first-party two-way GitHub Projects sync at no additional cost
- GitHub Projects v2 API gains view-config and field-color mutations (lifts the structural limitations)
- ntnxlab-ch onboarding requires cross-org tracking that single-forge tooling cannot serve
- Self-hosted Forgejo deployment becomes active (Phase 2 of `forge-abstraction.md`)

## References

- Research note: [`data/references/source_oversight_options.json`](../../data/references/source_oversight_options.json)
- Audit: [`docs/audit/2026-04-25-organize-self.md`](../audit/2026-04-25-organize-self.md)
- Operating model: arechste/git-organizer#396 (Pillar 4 — Govern)
- Related: [`forge-abstraction.md`](forge-abstraction.md) — the bigger forge-portability story
- Related: [`project-templates.md`](project-templates.md) — Project #12 (Global) remains; Project #15 retired
- Source issue: arechste/git-organizer#403
