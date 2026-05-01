# Promotion Protocol

How standing policies move from auto-memory → project rules → downstream-template distribution.

git-organizer is authoritative for git/gh/GitHub conventions. Standing policies that originate here may apply to other repos in the fleet, but **we don't push them to `~/dotclaude/`** — that would bloat the universal layer. Instead, we track them in `data/proposed-promotions.json` and let downstream repos opt in via templates or `additionalDirectories`.

## Lifecycle

```
Auto-memory feedback   →  Project rule              →  Proposed-promotion candidate  →  Distributed
(ephemeral correction)    (.claude/rules/*.md)         (data/proposed-promotions.json)   (consumer's CLAUDE.md/rules)
```

### Stage 1: memory feedback

A correction or surprising state surfaces in a session. Saved as `~/.claude/projects/<project>/memory/feedback_*.md`. Useful for ~one quarter; revisit at audit cycles.

**When to graduate:** the same correction surfaces in two unrelated sessions, OR the user references it as standing policy.

### Stage 2: project rule

Promote the feedback into `.claude/rules/standing-policies.md` (or distribute into existing rules). The rule is now loaded every session, no longer needs the memory entry.

After promotion, **delete** the corresponding memory file.

### Stage 3: proposed-promotion candidate

If the rule applies to other repos in the fleet, register it in `data/proposed-promotions.json` with:

- `name`: stable kebab-case identifier
- `source`: path to the rule file
- `appliesTo`: scope tags (`all-repos`, `github-repos`, `multi-machine`)
- `distributionMode`: see below
- `candidateConsumers`: explicit consumers
- `status: candidate`
- `rationale`: why this is portable

### Stage 4: distributed

When a consumer adopts the policy, update the entry to `status: promoted` with `promotedAt`. The mechanism depends on `distributionMode`:

| Mode | Mechanism |
|------|-----------|
| `rule-template` | Consumer copies the rule into its own `.claude/rules/`. Add `# Source: git-organizer/.claude/rules/<file>` header so it's discoverable as a tracked copy. |
| `additional-dir-include` | Consumer adds `~/airepos/claude/code/git-organizer` to `additionalDirectories` in its `.claude/settings.json`. Loads live; no copy. |
| `inline-snippet` | Consumer embeds the rule text into its CLAUDE.md or settings.json directly. For settings/permissions especially. |
| `convention-doc-link` | Consumer's rule references a `docs/conventions/*.md` doc. Light-touch, no enforcement layer copied. |

## Why no auto-push to dotclaude

dotclaude `~/.claude/` is the **universal layer**: every Claude Code session, every machine, every project. Pushing git/gh-specific standing policy there bloats sessions that don't touch git. Instead:

- **dotclaude home:** universal policy (Bash discipline, safety, code-of-conduct, tone)
- **git-organizer rules:** git/gh/GitHub-specific (commit format, branch naming, PR workflow, schedule governance)
- **proposed-promotions:** controlled distribution to specific consuming repos

The split keeps each layer minimal and the inheritance traceable.

## Retirement

A promotion is `retired` when:
- Tooling fully obsoletes it (e.g., `parallel-session-worktree-default` was retired-in-spirit when dotclaude#694/#695 shipped — but the principle stays as a rule)
- A new convention supersedes it (`replacedBy` field, mirrors the forge-index pattern)
- All candidate consumers have adopted alternative patterns

Retired promotions stay in `data/proposed-promotions.json` for audit history; they are not deleted.

## Validation

```bash
./tools/audit/build-promotion-index.sh --check
```

Verifies:
- JSON parses
- Schema valid (via `check-jsonschema`)
- Each `source` path exists in this repo

Runs in pre-commit when `data/proposed-promotions.json` is staged.

## Related

- `data/forge-index.json` — index of authored surfaces (skills, agents, conventions, references, hooks). Promotions are a complementary surface for "rules that should propagate."
- `docs/conventions/adoption.md` — how external repos consume our standards (forge-index OR `additionalDirectories`).
- `.claude/rules/inheritance-model.md` (or its post-relocation home) — full layered ownership model.
