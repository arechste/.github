# CLI Execution Standard

Canonical patterns for git/gh/GitHub operations in Claude sessions. Authored by git-organizer (domain expert). Full reference: `docs/conventions/claude-cli-execution.md`.

## Bash Discipline

- **One command per Bash tool call** — no `&&`, `||`, `;` ever
- **Dedicated tools first**: Read (not cat), Grep (not grep), Glob (not find), Write (not echo >)
- **No `#` in arguments**: use bare integers (`gh issue view 42`)
- **No variable capture**: `var=$(cmd)` is rejected — read output from Bash result
- **No `cd` prefix**: rely on session cwd

## Git Patterns

- `git switch` / `git restore` — never `git checkout`
- `git add path/to/file` — never `git add -A` or `git add .`
- `git commit -F- <<'EOF' ... EOF` — never `-m "$(cat ...)"`
- `git push -u origin HEAD` — first push sets upstream
- Branch workflow: 3 separate calls (`git switch main`, `git pull`, `git switch -c feat/thing`)

## gh CLI Patterns

- Write body to `.tmp/body.md` (Write tool), then `gh pr create --body-file ".tmp/body.md"`, then `rm .tmp/body.md`
- Use `--json` for structured output; use `--jq` only for simple field access (no string interpolation)
- MUST use `--cache <duration>` on all read-only `gh api` calls (prevents permission prompts from complex `--jq`). Default `--cache 1h`, use `--cache 5m` for fast-changing data. Omit only for write ops
- Use `--paginate` for list endpoints
- **Never** use `--jq` with `\(...)` string interpolation on `gh` subcommands — triggers obfuscation detector. Use `--json` only and read raw JSON output

## Working Directory

- Rely on session cwd — never prefix with `cd /path &&`
- Cross-repo reads: `git -C /path/to/repo <command>`
- Cross-repo writes: file a delegation issue

## Permission Matcher

These trigger approval prompts — avoid them:
- Compound operators: `&&`, `||`, `;`
- First-token issues: `var=`, `for`, `cd`
- Shell comment: `#` mid-argument
- Consecutive quotes: `--jq` with `\(...)` interpolation (obfuscation heuristic)
