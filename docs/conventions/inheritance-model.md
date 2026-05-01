# Rule Inheritance Model

Defines who owns which rules and how consuming repos load them.

## Ownership

| Layer | Owner | Location | Scope |
|-------|-------|----------|-------|
| **Universal** | dotclaude | `~/dotclaude/home/rules/` | All Claude Code sessions everywhere |
| **Git/GitHub** | git-organizer | `~/.../git-organizer/.claude/rules/` | Sessions in git-aware repos |
| **Repo-native** | Individual repo | `<repo>/.claude/rules/` | Sessions in that repo only |

## Layer Responsibilities

### Universal (dotclaude)

Rules that apply regardless of project context:
`coding-standards.md`, `tone-concise.md`, `safety.md`, `code-of-conduct.md`,
`claude-operations.md`, `execution.md`, `git-conventions.md`

Distributed via chezmoi from `~/dotclaude` → `~/.claude/` on each machine.

### Git/GitHub (git-organizer)

Rules specific to git workflow, GitHub API patterns, and multi-repo convention management. Currently in `.claude/rules/`:
`git-workflow.md`, `gh-api-usage.md`, `workflow-conventions.md`, `delegation.md`,
`agent-modes.md`, `permission-tiers.md`, `session-protocol.md`,
`monorepo-conventions.md`, `schedule-governance.md`, `review-handoff.md`,
`research-knowledge.md` (pointer; full loop in `.claude/skills/git-research/SKILL.md`),
`standing-policies.md`

Reference docs live in `docs/conventions/` and load on demand (not bundled into every session): `inheritance-model.md` (this doc), `secrets-audit.md`, `promotion-protocol.md`, etc.

Consumed by other repos via `additionalDirectories` in `.claude/settings.json`:
```json
"additionalDirectories": ["~/airepos/claude/code/git-organizer"]
```

Claude Code resolves the path and loads `.claude/rules/` files automatically.

### Repo-native

Rules that only make sense within a specific repo:
- `mac-organizer`: `plan-approval.md` (system-change gating, not applicable elsewhere)
- Any repo may add its own local rules; they stay local

## Anti-pattern: Rule Copies

Do NOT copy git-organizer rules into consuming repos. Copies diverge silently:
- A change to the canonical rule requires manual re-sync to every copy
- Copies inherit the wrong version at clone time
- Reviewers can't verify copy freshness

**Sentinel**: If a file carries the header `# Source: git-organizer/.claude/rules/<file>`, that file is a stale copy and should be replaced with `additionalDirectories` access.

## Bootstrap Fallback

`additionalDirectories` requires git-organizer to be cloned at the expected filesystem path.
For disconnected environments (CI containers, fresh machines, new team members):

1. **Fresh machine**: clone git-organizer as part of workspace setup:
   `git clone git@github.com:arechste/git-organizer.git ~/airepos/claude/code/git-organizer`
   This is already part of the standard `airepos/` layout documented in dotfiles.

2. **CI containers**: rules from `additionalDirectories` are not loaded in CI — sessions
   don't run in CI. CI scripts use shell scripts (`tools/`), not Claude Code rules.

3. **New repo bootstrap**: a repo can ship a one-time rule snapshot in `.claude/rules/`
   during setup, then add `additionalDirectories` and delete the snapshot once git-organizer
   is available. Document this in the repo's CLAUDE.md.

## Guard

No file outside `~/airepos/claude/code/git-organizer/.claude/rules/` should carry the
header `# Source: git-organizer/.claude/rules/`. If found, it is a stale copy to remediate.

The `dc:health` skill surfaces this drift. The detection command:
```bash
grep -r "Source: git-organizer" ~/.claude/ ~/dotclaude/ ~/dotfiles/ ~/airepos/
```
