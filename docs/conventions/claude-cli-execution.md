# Claude CLI Execution Standard

Canonical patterns for how Claude executes git, gh CLI, and GitHub API operations. This is the authoritative reference from git-organizer (domain expert). A concise distributable rule is in `claude-cli-execution-rule.md`.

**Audience**: Claude sessions across all repos. Skills consume a structured version via `@shared/cli-execution.md`.

## 1. Working Directory Strategy

Claude Code maintains a persistent working directory (cwd) for the session. All commands execute relative to this cwd.

### Rules

- **Rely on session cwd** — do not prefix commands with `cd /path/to/repo`
- **Never chain**: `cd ~/repo && git status` is forbidden (compound operator)
- **Cross-repo reads**: use `git -C /path/to/repo <command>` for one-off inspection of another repo
- **Cross-repo writes**: file a delegation issue instead of operating on another repo
- **Worktrees**: Claude Code sets cwd automatically when entering a worktree

### Patterns

| Scenario | Canonical | Anti-pattern |
|----------|-----------|-------------|
| Check status | `git status` | `cd ~/repo && git status` |
| Inspect other repo | `git -C /path/to/repo log --oneline -5` | `cd /path/to/repo && git log` |
| Multi-repo work | Delegation issue or separate session | `cd` between repos in one session |

## 2. Git Command Patterns

Each operation below is a **single Bash tool call**. Never combine operations.

### Branch operations

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Switch to main | `git switch main` | Never `git checkout main` |
| Pull latest | `git pull` | Separate call from switch |
| Create branch | `git switch -c feat/description-N` | From current HEAD |
| List branches | `git branch -a` | |
| Delete branch | `git branch -d feat/description-N` | Safe delete (checks merge) |
| Switch branch | `git switch feat/description-N` | Never `git checkout` |

**Branching workflow** (3 separate Bash calls):
```
Call 1: git switch main
Call 2: git pull
Call 3: git switch -c feat/thing-42
```

### Staging and committing

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Stage specific file | `git add path/to/file.ext` | Always name files explicitly |
| Stage multiple files | `git add file1.ext file2.ext` | Space-separated, or multiple calls |
| View staged changes | `git diff --cached` | |
| View unstaged changes | `git diff` | |
| Commit with heredoc | See below | Never use `-m "$(cat ...)"` |

**Commit pattern** (heredoc via `-F-`):
```bash
git commit -F- <<'EOF'
feat(scope): description of change (#42)

Optional body explaining why, not what.

Co-Authored-By: Claude/Opus/Auditor@hostname <noreply@anthropic.com>
EOF
```

**Why `-F-` over `-m`**: heredoc allows multi-line messages without shell escaping issues. The `<<'EOF'` (quoted) prevents variable expansion.

### History and inspection

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Recent log | `git log --oneline -10` | |
| Full log entry | `git log -1 --format=fuller` | |
| Diff between branches | `git diff main..HEAD` | |
| Diff of specific file | `git diff main -- path/to/file` | |
| Blame | `git blame path/to/file` | |
| Show commit | `git show abc1234` | |

### Remote operations

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Fetch | `git fetch origin` | Separate from rebase/merge |
| Push (first time) | `git push -u origin HEAD` | Sets upstream tracking |
| Push (subsequent) | `git push` | After upstream is set |
| Pull | `git pull` | Only on tracking branch |
| Rebase on main | `git rebase origin/main` | After `git fetch origin` (separate call) |

### File restoration

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Unstage file | `git restore --staged path/to/file` | Never `git checkout -- file` |
| Discard changes | `git restore path/to/file` | Never `git checkout -- file` |
| Restore from commit | `git restore --source=abc1234 path/to/file` | |

### Tags

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Create annotated tag | `git tag -a v1.4.0 -m "description"` | |
| Push tag | `git push origin v1.4.0` | |
| List tags | `git tag -l` | |

### Stash

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Stash changes | `git stash push -m "description"` | |
| List stashes | `git stash list` | |
| Apply latest | `git stash pop` | |

## 3. gh CLI Patterns

### Content creation (PR, issue, release)

**Pattern**: Use Write tool to create content in `.tmp/`, reference via `--body-file`, then delete.

```
Step 1 (Write tool):  Write content to .tmp/pr-body.md
Step 2 (Bash tool):   gh pr create --title "feat(scope): description" --body-file ".tmp/pr-body.md"
Step 3 (Bash tool):   rm .tmp/pr-body.md
```

Always delete **only the specific file you created** after the consuming command succeeds. Never bulk-delete `.tmp/` contents — other skills or processes may have files there. Never use heredoc or shell substitution for gh body content.

### Pull requests

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Create PR | `gh pr create --title "type(scope): desc" --body-file ".tmp/pr-body.md"` | Write body file first |
| View PR | `gh pr view 42 --json title,body,state,reviews` | Use `--json` for structured data |
| List PRs | `gh pr list --json number,title,state,labels` | |
| Merge PR | `gh pr merge 42 --squash --delete-branch` | Single command |
| Check CI | `gh pr checks 42` | |
| Add labels | `gh pr edit 42 --add-label "type/feature"` | |

### Issues

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Create issue | `gh issue create --title "type(scope): desc" --body-file ".tmp/issue-body.md"` | Write body file first |
| View issue | `gh issue view 42 --json title,body,state,labels` | |
| List issues | `gh issue list --json number,title,state,labels` | Raw JSON, no `--jq` interpolation |
| Close issue | `gh issue close 42` | |
| Edit labels | `gh issue edit 42 --add-label "status/ready"` | |
| Assign | `gh issue edit 42 --add-assignee @me` | |

### Labels

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| List labels | `gh label list --json name,color,description` | |
| Create label | `gh label create "type/feature" --color "1D76DB" --description "New functionality"` | |
| Edit label | `gh label edit "type/feature" --color "1D76DB"` | |

### Releases

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| Create release | `gh release create v1.4.0 --title "v1.4.0" --notes-file ".tmp/release-notes.md"` | Write notes first |
| List releases | `gh release list` | |
| View release | `gh release view v1.4.0` | |

### Workflows

| Operation | Canonical command | Notes |
|-----------|------------------|-------|
| List runs | `gh run list --limit 5` | |
| View run | `gh run view 12345` | |
| View logs | `gh run view 12345 --log-failed` | |
| Rerun | `gh run rerun 12345` | |

## 4. GitHub API Patterns

### When to use `gh api` vs `gh` subcommands

| Use `gh` subcommands | Use `gh api` |
|----------------------|-------------|
| PR, issue, release, label, run, repo operations | Endpoints without a subcommand (vulnerability-alerts, traffic, Projects v2) |
| Human-readable output needed | Structured JSON processing needed |
| Standard CRUD operations | GraphQL queries |
| | Bulk operations in scripts |

### API call patterns

| Pattern | Command | Notes |
|---------|---------|-------|
| GET with filter | `gh api repos/OWNER/REPO/labels --jq '.[].name'` | Server-side filter |
| GET with cache | `gh api repos/OWNER/REPO/labels --cache 1h` | Read-only, reduces rate limit |
| Paginated list | `gh api repos/OWNER/REPO/issues --paginate --jq '.[].title'` | All pages |
| PUT (enable) | `gh api repos/OWNER/REPO/vulnerability-alerts -X PUT` | |
| POST with data | `gh api repos/OWNER/REPO/labels -f name=bug -f color=d73a4a` | |
| GraphQL | `gh api graphql -f query='{ viewer { login } }'` | |

### Rate limit awareness

- Check before bulk operations: `gh api /rate_limit --jq '.rate.remaining'`
- Write operations: minimum 1 second between requests
- Read-only bulk: 0.3-0.5 seconds between requests
- MUST use `--cache <duration>` on all read-only `gh api` calls — prevents permission prompts from complex `--jq` expressions. Default `--cache 1h`, use `--cache 5m` for fast-changing data
- In scripts: use `gh_rate_limit_check` and `gh_api_retry` from `tools/lib/gh-helpers.sh`

### `--jq` obfuscation safety

Claude Code's permission matcher flags `--jq` expressions containing string interpolation (`\(...)`) as "consecutive quote characters at word start (potential obfuscation)". This is a **platform-level heuristic** that cannot be overridden by `settings.json`.

**Affected commands**: any `gh` subcommand with complex `--jq` — `gh issue list`, `gh pr list`, `gh repo list`, `gh api` (without `--cache`).

**Safe patterns** (simple `--jq`, no string interpolation):
```
gh issue list --json number,title,labels --jq '.[].title'
gh api repos/OWNER/REPO/labels --cache 1h --jq '.[].name'
```

**Triggers the prompt** (`--jq` with `\(...)` interpolation):
```
gh issue list --json number,title --jq '.[] | "\(.number) \(.title)"'
```

**Workaround for Bash tool calls**: use `--json` without `--jq` and read the raw JSON output directly from the Bash tool result. Claude can parse JSON natively.

```
# Instead of complex --jq interpolation:
gh issue list --json number,title,state,labels

# gh api: --cache sidesteps the permission prompt entirely:
gh api repos/OWNER/REPO/issues --cache 1h --jq '.[] | "\(.number) \(.title)"'
```

**In shell scripts**: no restriction — scripts run outside the permission matcher.

### Environment variables

Set these in scripts (automatically handled by `gh-helpers.sh`):

| Variable | Value | Purpose |
|----------|-------|---------|
| `NO_COLOR` | `1` | Suppress ANSI escapes that break JSON parsing |
| `GH_NO_UPDATE_NOTIFIER` | `1` | Suppress update notices on stderr |
| `GH_API_VERSION` | `2022-11-28` | Pin API version for stability |

## 5. Bash Discipline

### Two contexts, different rules

| Context | Compound operators | `|| true` guards | Purpose |
|---------|-------------------|-----------------|---------|
| **Shell scripts** (`set -euo pipefail`) | Allowed inside scripts | Required for grep/test | Deterministic execution |
| **Claude Bash tool calls** | **Forbidden** | Not applicable | One command per call |

Scripts in `tools/` use `&&`, `||`, pipes, and loops freely — that's correct shell scripting. The restrictions below apply only to Claude's Bash tool calls.

### Rules for Bash tool calls

1. **One command per call** — no exceptions
2. **No compound operators**: `&&`, `||`, `;` trigger the permission matcher
3. **No variable assignment as first token**: `var=$(command)` is rejected
4. **No `for` loops**: `for x in ...` is rejected as first token
5. **No `#` in arguments**: permission matcher treats mid-word `#` as shell comment
6. **No `cd` prefix**: rely on session cwd, use `git -C` for other repos
7. **Dedicated tools first**: Read (not `cat`), Grep (not `grep`), Glob (not `find`), Write (not `echo >`)
8. **Multi-line content**: Write tool to `.tmp/`, then reference via `--body-file` or `-F-`

### What IS allowed in a single Bash call

- Pipes: `git log --oneline -10` (single command with flags)
- Heredoc stdin: `git commit -F- <<'EOF' ... EOF`
- Redirects: `gh api /rate_limit --jq '.rate.remaining'` (jq flag, not a pipe)
- Script execution: `./tools/audit/fetch-repos.sh` (compound logic is inside the script)

## 6. Permission Matcher Awareness

The Claude Code Bash permission matcher evaluates each Bash tool call. These patterns trigger an approval prompt:

| Trigger | Example | Why it's rejected |
|---------|---------|-------------------|
| `&&` operator | `git fetch && git rebase` | Compound command |
| `\|\|` operator | `command \|\| true` | Compound command |
| `;` separator | `git add .; git commit` | Compound command |
| Variable assignment first | `result=$(git status)` | Unrecognized first token |
| `for` loop first | `for f in *.md; do` | Unrecognized first token |
| `cd` prefix | `cd ~/repo && git log` | Compound command (and unnecessary) |
| `#` mid-argument | `gh issue view #42` | Treated as shell comment |
| Consecutive quotes | `--jq '.[] \| "\(.field)"'` | Obfuscation heuristic (escaped quotes at word start) |

### How settings.json patterns work

Permission patterns match the first token of the Bash command:

```json
{
  "allow": [
    "Bash(git:*)",          // Matches: git status, git push, git log
    "Bash(gh:*)",           // Matches: gh pr create, gh api, gh issue list
    "Bash(./tools/*)"    // Matches: ./tools/audit/fetch-repos.sh
  ],
  "deny": [
    "Bash(git push --force:*)",   // Blocks force push
    "Bash(git reset --hard:*)"    // Blocks hard reset
  ]
}
```

The first token is extracted before `:` — so `git status` matches `Bash(git:*)`. Compound operators introduce a second command whose first token may not be in the allow list.

### Avoiding prompts

- Split compound operations into separate Bash calls
- Use issue numbers as bare integers: `gh issue view 42` (not `#42`)
- Capture command output by reading it from the Bash tool result, not via `$()`
- Use Write tool for multi-line content instead of heredoc-based file creation
- Avoid `--jq` with string interpolation (`\(...)`) on `gh` subcommands — use `--json` only and read raw JSON output (see below)

## Cross-references

- Distributable rule: `docs/conventions/claude-cli-execution-rule.md`
- Shared resource for skills: `skills/shared/cli-execution.md`
- Git workflow rules: `rules/git-workflow.md`
- gh API usage: `rules/gh-api-usage.md`
- Commit format: `docs/conventions/commit-format.md`
- Temp files: `docs/conventions/temp-files.md`
