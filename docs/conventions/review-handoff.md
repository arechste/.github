# Review Handoff

When Claude should ask for review, how deep that review needs to be, and how the "merge" instruction interacts with a review gap. Applies to `pr-only` repos where Claude is forbidden to self-merge regardless of CI state.

This convention fills the gap between:

- `permission-matrix.md` / `permission-tiers.md` — what Claude is *allowed* to do
- `auto-merge-policy.md` — when machine-driven auto-merge is appropriate
- `change-verification.md` — post-merge verification bars

## Review-Depth Tiers (Q1)

Claude classifies every PR into one of four tiers at creation time. The classification is deterministic — driven by file-path patterns, scope size, and label triggers below.

| Tier | Signal | Reviewer effort |
|------|--------|-----------------|
| **Skim** | Doc-only change, single-file rule update, regenerated index, changelog, comment typo | Read PR title + summary, click merge |
| **Read** | Convention change, new rule, multi-file refactor under one concept, label/template update, new skill | Read the PR body's "what changed" section + skim the diff |
| **Verify** | Touches a sensitive path (see list below); cross-repo delegation; first use of a new pattern; CI workflow files; label-sync or fleet-wide sync | Open the files Claude flagged, run the commands listed in "What I already verified", then decide |
| **Block** | Destructive or irreversible action; scope ambiguity Claude cannot resolve; requires human judgment Claude cannot simulate | Have the conversation first; PR body explains what decision is needed |

### Tier classification rules

Apply the **highest-matching tier**.

**Skim** — ALL of the following must be true:
- All changed files have extension `.md` OR are known-regenerated outputs (`data/forge-index.json`, `data/reference-index.json`, `data/label-audit-report.md`, `CHANGELOG.md`)
- Diff line count ≤ 100 (approximated as `git diff --stat | tail -1 | awk '{print $4}'`)
- No sensitive-path match (see below)
- No cross-repo or delegation label on the PR

**Read** — any of:
- Changed file is in `docs/conventions/`, `docs/references/`, `.claude/rules/`, `~/dotclaude/home/rules/` (convention or rule change)
- New skill file added (`home/skills/*/SKILL.md`, `.claude/skills/*/SKILL.md`)
- Multi-file refactor (≥2 non-docs files changed, all under one logical concept)
- Label or PR template change (`.github/`)
- `data/repo-inventory.json` change that doesn't touch sensitive fields (`autonomyLevel`, `autoMergeDefault`)

**Verify** — any sensitive-path match OR:
- `data/repo-inventory.json` change to `autonomyLevel`, `autoMergeDefault`, or `mergePolicy` fields
- `data/delegated-issues.json` or `data/delegation-inbox.json` change
- Cross-repo delegation label on the PR (`delegated`, `delegation/*`)
- First use of a new pattern (Claude's judgment; must be stated explicitly in the PR body)

**Block** — any of:
- Destructive action in the change (schema deletion, label deletion affecting >3 repos, data file truncation)
- Scope grew past what the issue described and the delta isn't captured in an updated checklist
- Security-sensitive file changed that is NOT already on the sensitive-paths list (unexpected; needs human to define the bar)
- Claude's own confidence in correctness is low (must be stated explicitly)

### Sensitive-paths list

Any file matching these patterns triggers at minimum **Verify**:

```
.claude/settings.json
.claude/settings.local.json
~/.claude/settings.json
~/.claude/settings.local.json
.github/workflows/*.yml
.github/workflows/*.yaml
renovate.json
renovate/*.json
tools/labels/*.sh
tools/maintenance/delegate-issue.sh
data/repo-inventory.json           # autonomyLevel / autoMergeDefault / mergePolicy fields only
*secrets*
*credentials*
*token*
*.pem
*.key
```

This list is the project-level minimum. Consuming repos may extend it in their own `review-handoff.md`.

## PR Body Review-Tier Banner (Q2)

Claude MUST include a review-tier banner at the top of every PR body. The banner is the first element, before the summary.

### Skim

```
**Review tier**: ✅ Skim — doc-only / regenerated output
```

### Read

```
**Review tier**: 🔵 Read
**What changed**: <one sentence describing the convention or rule update>
```

### Verify

```
**Review tier**: 🟡 Verify
**What to look at**: `<file1>`, `<file2>` — <one sentence per file on what to check>
**What I already verified**: <bullet list of commands run and their outcome>
```

### Block

```
**Review tier**: 🔴 Block — human judgment required
**Decision needed**: <one sentence stating the decision>
**Context**: <brief background so the human can decide without re-reading the issue>
```

The banner is mandatory. Skills that call `gh pr create` are responsible for computing the tier and writing the banner into the PR body. The PR template provides the skeleton; the skill fills the tier fields.

## When the User Types "merge" Without Review (Q3)

**Stance: trust-but-warn for Skim/Read; strict for Verify/Block.**

### Skim / Read tier

Claude executes the merge on instruction. Before running `gh pr merge`, Claude echoes once in the conversation:

```
Merging PR #<N> (<tier> tier — <brief rationale>). No explicit review signal noted.
```

This warning lands in the conversation log without blocking. It is not a prompt for confirmation.

### Verify tier

Claude does NOT merge on a bare "merge" instruction. Instead, Claude asks one targeted question:

```
PR #<N> is Verify tier. Before I merge, did you check <specific item from "What to look at">?
```

If the user confirms ("yes", "checked", "looks good"), Claude merges. The confirmation count resets per session (asking once is enough).

If the user overrides with "merge anyway" or "skip review", Claude merges and logs:

```
Merging PR #<N> (Verify tier) on explicit override. Review of <item> not confirmed.
```

### Block tier

Claude does NOT merge, period. The PR body explains the decision needed. Claude's response to "merge" on a Block-tier PR is:

```
PR #<N> is Block tier. Merging requires a human decision on: <item from banner>.
Once you've decided, let me know how to proceed and I'll implement the change.
```

### Implementation note for the `work` skill

The tier is encoded in the PR body banner. A skill re-reading the PR to decide merge behavior should parse the `**Review tier**` line. Pattern: `🔴` → Block (never merge); `🟡` → Verify (ask once); `🔵` or `✅` → Skim/Read (warn-and-merge).

## When Claude Proactively Requests Review (Q4)

Claude pauses before opening the PR and asks for review when any of the following triggers fire. These are **pause-and-ask** triggers, not just tier-classification triggers — they require an explicit go-ahead before the PR is opened.

| Trigger | Action |
|---------|--------|
| Sensitive-path match AND the change is not an expected routine update (e.g., an unplanned `settings.json` change) | Describe what changed and why; ask for review approval before `gh pr create` |
| Scope grew past the issue's acceptance criteria | Summarize the delta; ask if the expansion is intended or should be split into a follow-up issue |
| New cross-cutting pattern introduced (one that other code will follow) | Name the pattern; explain why it was needed; confirm before PR |
| User previously corrected a similar change in this session | Reference the correction; explain why this case is different; confirm before PR |
| `Block` tier computed | Describe the decision needed; do not open the PR until the human resolves it |

Proactive requests are targeted — one specific question, not a general "want me to open the PR?". The default is to open the PR; pause only when a trigger fires.

## Relationship to Existing Conventions (Q5)

```
permission-tiers.md          — what Claude is allowed to do (autonomyLevel gate)
       ↓
review-handoff.md            — human review handoff (this file)
       ↓
auto-merge-policy.md         — when machine-driven merge is OK (autonomyLevel: full, type prefix, escape hatches)
       ↓
change-verification.md       — post-merge verification bar (adoption, dry-run, chezmoi apply)
```

- **permission-tiers.md**: defines `pr-only` (Claude opens PRs; human merges). This convention defines what "human merges" means in practice for `pr-only` repos.
- **auto-merge-policy.md**: covers `full`-tier repos where the skill can poll-and-merge. This convention covers the `pr-only` case where it cannot.
- **change-verification.md**: applies after the merge regardless of who merges and regardless of review depth. The two conventions are independent axes.

## References

- `docs/conventions/auto-merge-policy.md` — when machine-driven merge is appropriate
- `docs/conventions/permission-matrix.md` — action-by-autonomy-level matrix
- `.claude/rules/permission-tiers.md` — autonomy level definitions
- `docs/conventions/change-verification.md` — post-merge verification bars
- Operating-model charter: issue #396 (Pillar 4 — Govern)
