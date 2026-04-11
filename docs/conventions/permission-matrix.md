# Permission Matrix

Full action-by-autonomy-level matrix for Claude operations across repos.

See `rules/permission-tiers.md` for autonomy level definitions and decision criteria.

## Action Matrix

| Action | read-only | assist | pr-only | full |
|--------|-----------|--------|---------|------|
| git status / log / diff | Auto | Auto | Auto | Auto |
| gh api GET queries | Auto | Auto | Auto | Auto |
| git add / commit (branch) | Blocked | Blocked | Auto | Auto |
| git push (feature branch) | Blocked | Blocked | Auto | Auto |
| gh pr create | Blocked | Blocked | Auto | Auto |
| gh issue create | Blocked | Approval | Auto | Auto |
| gh label create / edit | Blocked | Approval | Auto | Auto |
| gh pr merge (CI passes) | Blocked | Blocked | Blocked | Auto |
| gh pr merge (CI fails) | Blocked | Blocked | Blocked | Blocked |
| gh label delete | Blocked | Blocked | Approval | Approval |
| git push to main | Blocked | Blocked | Blocked | Blocked |
| gh repo archive | Blocked | Blocked | Blocked | Approval |
| git reset --hard | Blocked | Blocked | Blocked | Blocked |
| git push --force to main | Blocked | Blocked | Blocked | Blocked |
| gh repo delete | Blocked | Blocked | Blocked | Blocked |

**Auto** = Claude can do this without asking.
**Approval** = Claude must ask the human first.
**Blocked** = Never allowed, regardless of mode or context.

## Permission Boundaries by Mode

Mode-specific view (from `rules/agent-modes.md`):

| Action | Auto | Approval needed | Blocked |
|--------|------|-----------------|---------|
| git status/log/diff | Yes | — | — |
| git switch -c (new branch) | Yes | — | — |
| git add / commit (on branch) | Yes | — | — |
| git push (to feature branch) | Yes | — | — |
| gh pr create | Yes | — | — |
| gh issue create | Yes | — | — |
| gh label create/edit | Yes | — | — |
| gh pr merge (if CI passes) | Per autonomy level | If conflicts | — |
| git push to main | — | Always | — |
| gh label delete | — | Always | — |
| git reset --hard | — | — | Blocked |
| git push --force to main | — | — | Blocked |
| gh repo delete | — | — | Blocked |
| gh repo archive | — | Always | — |

## Permission Shadowing

Claude Code uses **shadowing, not union** for project permissions. When a project defines `permissions.allow` in `.claude/settings.json`, it becomes the authoritative list — the global `~/.claude/settings.json` allow list is NOT merged in. Any project that defines `permissions.allow` must repeat commonly-needed commands.

## Recommended .claude/settings.json Patterns

### For pr-only repos (most common)

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(gh:*)",
      "Bash(jq:*)",
      "Bash(shellcheck:*)",
      "Bash(rm:.tmp/*)",
      "Bash(rm -f:.tmp/*)",
      "Bash(mkdir:*)",
      "Bash(mktemp:*)"
    ],
    "deny": [
      "Bash(git push --force:*)",
      "Bash(git push -f:*)",
      "Bash(git reset --hard:*)",
      "Bash(gh repo delete:*)",
      "Bash(gh repo archive:*)",
      "Bash(gh auth login:*)",
      "Bash(gh auth refresh:*)"
    ]
  }
}
```

### For read-only repos

```json
{
  "permissions": {
    "allow": [
      "Bash(git:*)",
      "Bash(gh:*)",
      "Bash(jq:*)"
    ],
    "deny": [
      "Bash(git add:*)",
      "Bash(git commit:*)",
      "Bash(git push:*)",
      "Bash(git reset:*)",
      "Bash(gh pr create:*)",
      "Bash(gh pr merge:*)",
      "Bash(gh issue create:*)",
      "Bash(gh repo delete:*)"
    ]
  }
}
```

### For full repos

Same as pr-only. The `Bash(gh:*)` broad pattern already covers `gh pr merge`.
No additional allow entries needed — just note that full repos omit `gh pr merge` from deny.
