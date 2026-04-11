# Temporary File Conventions

## Project-Local `.tmp/` Directory

Every project should have a `.tmp/` directory (gitignored) for temporary files created by scripts and tools. This follows the WAT framework pattern from mac-organizer.

```
.tmp/          # Project-local temp files (gitignored)
.tmp/.gitkeep  # Keep the directory in git
```

### `.gitignore` entry

```gitignore
.tmp/*
!.tmp/.gitkeep
```

## Rules

1. **Never use predictable `/tmp/` paths** like `/tmp/my-script-output.json`. These cause:
   - Permission issues when multiple users run the same script
   - Race conditions when running concurrently
   - Orphaned files that never get cleaned up

2. **Always use `mktemp`** for randomized names. In git-organizer, use the `mktemp_local` helper:
   ```bash
   local tmp
   tmp=$(mktemp_local "prefix")
   cleanup_on_exit "${tmp}"
   ```

3. **Always register for cleanup** using `cleanup_on_exit` (trap-based, guaranteed even on error):
   ```bash
   cleanup_on_exit "${tmp1}" "${tmp2}"
   ```
   Never rely on manual `rm -f` calls — they're skipped on early exit.

4. **The `${FILE}.tmp` suffix pattern is different and fine**. Writing to `${INVENTORY_FILE}.tmp` then `mv` to `${INVENTORY_FILE}` is an atomic write pattern, not temp file management.

## git-organizer Helpers

`tools/lib/gh-helpers.sh` provides:

| Function | Purpose |
|----------|---------|
| `mktemp_local "prefix"` | Create temp file in `.tmp/` with randomized name |
| `cleanup_on_exit "$file"` | Register file for automatic cleanup on EXIT |

## Cleanup Strategy

Temp file cleanup happens at two layers:

### Immediate: delete after use

Delete the temp file as soon as the consuming command succeeds. Once `gh pr create`, `gh issue create`, or `git commit` returns 0, the file has served its purpose:

For Claude Code Bash tool calls, use the Write tool to create the file, then a single `gh` command, then delete the temp file:

1. Write tool → `.tmp/pr-body.md`
2. `gh pr create --body-file ".tmp/pr-body.md"`
3. `rm .tmp/pr-body.md`

Always delete **only the specific file you created** after the consuming command succeeds. Never bulk-delete `.tmp/` contents (`rm .tmp/*`, `find .tmp -delete`) — other skills or concurrent processes may have files there. The session-end hook is a safety net for crashes, not the primary cleanup mechanism.

For shell scripts, use `mktemp_local` with trap-based cleanup:

```bash
body_file=$(mktemp_local "pr-body")
cat > "${body_file}" <<'EOF'
...
EOF
gh pr create --body-file "${body_file}"
# cleanup_on_exit handles removal via trap
```

Do NOT chain commands with `&&` in Claude Code Bash calls — the permission matcher rejects compound commands.

### Safety net: session-end hook

The `session-end.sh` hook (in dotclaude) is a safety net that cleans `.tmp/` files:
1. All unlocked files (not just mktemp-patterned — includes semantic names like `pr-body.md`)
2. Files older than 24h (unconditional — handles crashed/interrupted sessions)

This catches files orphaned by killed scripts or interrupted sessions. It is NOT the primary cleanup mechanism — always delete temp files after successful use.

**Note for dotclaude**: The hook must clean ALL files in `.tmp/` except `.gitkeep`, not just `*.??????` mktemp-patterned files. Claude Code's Write tool creates semantic names (`pr-body.md`, `issue.md`) that don't match the mktemp pattern.

## For Governed Repos

Repos governed by git-organizer should:
- Add `.tmp/` to `.gitignore`
- Use `mktemp` (system or project-local) instead of hardcoded paths
- Avoid `/tmp/` in scripts — use project-local `.tmp/` when possible
- The `git-no-tmp-hardcoded` check in `data/repo-health-checks.json` flags violations
