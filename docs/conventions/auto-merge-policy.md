# Auto-Merge Policy

When the merge button is a ceremony vs. a gate, and how skills should decide per-PR.

> **TL;DR**: PRs are the gate (CI visibility, diff review, revert point). The merge
> click is a ceremony — automate it where blast radius is low, keep it manual where
> blast radius is high. Decision lives in the `work` skill, keyed on the branch-type
> prefix already encoded in `{hostname}/{type}/{N}-{slug}`.

## Framing: PR as gate, not ceremony

The rule "no direct commits to main, all changes via PR" (`.claude/rules/git-workflow.md`)
exists because the PR itself is the gate:

- **CI visibility** — workflows run against the PR head and surface pass/fail
- **Diff review** — a human (or reviewing agent) can inspect the proposed change
- **Revert point** — squash-merge produces a single commit that's trivially revertible

**None of those require the merge button to be clicked manually.** Equating "PR
discipline" with "manual merge required" is a misread: the discipline is the
*existence* of the PR, not the ceremony of the click. Skills and conventions should
make this framing explicit so future agents don't assume manual merge is load-bearing.

## Platform constraint (free tier)

All four team repos (`dotfiles`, `dotclaude`, `git-organizer`, `mac-organizer`) are
**private on GitHub Free**. Branch protection, required status checks, and native
auto-merge are unavailable:

```
PATCH /repos/{owner}/{repo}  { "allow_auto_merge": true }
→ 403: Upgrade to GitHub Pro or make this repository public
```

Setting `allow_auto_merge` at the repo level is a no-op in this state, and
`gh pr merge --auto` requires it. So on free tier the choice is binary:

- **Manual merge** (current default) — `gh pr merge --squash --delete-branch <N>`
- **Local poll-and-merge** — skill polls `gh pr checks` until green, then merges

Both produce identical history on main. The difference is who clicks.

## Three paths (evaluated)

| Path | Cost | Unlocks | Tradeoff |
|------|------|---------|----------|
| Upgrade to GitHub Pro | ~$4/mo per seat | Branch protection, `gh pr merge --auto`, required checks | Recurring cost for a feature we can replicate locally |
| Make select repos public | $0 | Branch protection, Dependabot on private hosts of same content | Exposes per-machine detail in dotfiles, fleet inventory in mac-organizer; SOPS-encrypted blobs stay safe but surrounding context leaks |
| Local poll-and-merge helper | $0 | Auto-merge semantics via `gh pr checks` polling in the `work` skill | Skill owns the wait loop; no GitHub-side UI checkbox |

### Recommendation

**Path 3 (local poll-and-merge helper).** It covers all four repos on free tier
with no recurring cost, and is the only path that actually addresses the stated
problem — PRs merging manually as a ceremony.

Reasoning:

- **Path 3 is the cheapest and most aligned with our stack.** We already run
  local-first CI (`tools/ci/local-check.sh`, pre-commit), skills already execute
  locally, and `gh pr checks` gives the same pass/fail signal a branch-protection
  rule would enforce. No recurring cost, no new platform dependency.
- **Path 1 buys a UI feature we don't need.** Branch protection is primarily a
  *multi-contributor* guardrail. We are a single maintainer with agent assistance —
  the guardrail is already the `work` skill's own discipline plus local pre-commit.
- **Path 2 is out of scope for this policy.** Making private repos public is a
  separate decision with its own drivers (Dependabot, external contribution) and
  its own tradeoffs. It does not unlock auto-merge for the repos that need it
  most (`dotfiles`, `mac-organizer` stays private either way), so it's not a
  substitute for path 3 and shouldn't be coupled to this decision. Repos stay
  private until explicitly decided otherwise.

## Per-repo stance

Independent of which path is chosen, each repo declares whether auto-merge is
appropriate for its PR flow:

| Repo | Auto-merge default | Rationale |
|------|---------------------|-----------|
| `dotfiles` | **yes** | Solo maintainer; `chezmoi apply` is an explicit user action (merge ≠ deploy); mostly chore/docs PRs; revert is cheap both in git and operationally |
| `dotclaude` | **yes** | Config/skills/rules; changes take effect on next `chezmoi apply`; low blast radius; easy to revert |
| `git-organizer` | **conditional** | Yes for docs/convention housekeeping; manual for rule changes that ripple into consuming repos via `additionalDirectories` |
| `mac-organizer` | **no** | Fleet-wide blast radius; boot-critical scripts (`nixadmin-setup --remove`, FileVault); revert is cheap in git but operationally expensive — machines already touched |

The repo-level stance is the ceiling; the per-PR decision (below) can drop below it
but never above. `mac-organizer` stays manual regardless of PR type.

## Per-PR decision matrix

Even in a repo where auto-merge is enabled at the repo level, the `work` skill
decides per-PR using the branch-type prefix:

| Type prefix | Auto-merge | Notes |
|-------------|------------|-------|
| `chore` | yes | Tooling, dependencies, cleanup |
| `docs` | yes | README, convention docs, ADRs |
| `test` | yes | Test additions/fixes |
| `ci` | yes | CI-only changes (workflow files) |
| `refactor` | no | Behavior-preserving but wide-reaching; human review |
| `feat` | no | New behavior; human review |
| `fix` | **conditional** | `priority/low` + no convention touch → yes; otherwise no |
| `hotfix` | no | Always human-reviewed — hotfix implies production risk |
| `release` | no | Release PRs are the ceremony; never auto |
| `security` | no | Always human-reviewed regardless of scope |
| `claude` | conditional | Agent-originated work; same rules as the type it would have carried |

The branch format `{hostname}/{type}/{N}-{slug}` already encodes the type; no new
metadata is needed.

**Escape hatches**:

- `no-auto-merge` label on the PR → always manual, regardless of type
- `auto-merge` label on the PR → forces auto even where the type default is no;
  requires a human to apply, which is itself the gate
- Existing `agent/*` labels still take precedence: `agent/human` never auto-merges

## Skill contract

The `work` skill (owned by `dotclaude`) is the surface where this policy takes
effect. It must:

1. Read the repo's auto-merge default (new field in `data/repo-inventory.json`:
   `autoMergeDefault: "yes" | "no" | "conditional"`)
2. Look up the PR's branch-type prefix
3. Apply the decision matrix above, honoring escape-hatch labels
4. If the decision is **auto-merge**: after `gh pr create`, poll `gh pr checks
   <N> --watch` (blocks until terminal), then `gh pr merge --squash --delete-branch <N>`
5. If the decision is **manual**: report the PR URL and exit; leave merge to the
   human

Cap the poll with a wall-time ceiling (suggested: 15 min) and fall back to manual
with a clear message on timeout. `gh pr checks --watch` already exits non-zero on
any check failure, so a failed CI run naturally falls back to manual without a
forced merge.

## Related conventions

- `docs/conventions/merge-completion.md` — post-merge cleanup (`--delete-branch`)
- `.claude/rules/git-workflow.md` — merge strategy (squash only, linear history)
- `.claude/rules/permission-tiers.md` — `autonomyLevel: full` is a prerequisite for
  auto-merge at the repo level; `pr-only` repos never auto-merge regardless of type
- `docs/conventions/change-verification.md` — verification bar is unaffected by who
  clicks merge; the bar still applies

## Open questions (deferred)

- Should `autoMergeDefault` live in `data/repo-inventory.json` or in a new
  `data/automerge-policy.json`? Inventory keeps per-repo config co-located; a
  separate file makes the policy easier to audit as a set.
- Do we want a session-start report of PRs that auto-merged since last session?
  Low priority — `gh pr list --state merged --limit 20` already answers this.
- Rollout order: adopt in `dotclaude` first (self-hosting the skill), then
  `git-organizer`, then `dotfiles`. `mac-organizer` stays out of scope.
