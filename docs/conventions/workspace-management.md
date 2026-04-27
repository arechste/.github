# Workspace Management

Conventions for managing local git clones across machines.

## Directory Layout

Two directory trees are scaffolded by dotfiles (via chezmoi) on all machines:

| Tree | Path | Purpose |
|------|------|---------|
| repos | `~/repos/github.com/<owner>/<repo>` | Human-initiated work |
| airepos | `~/airepos/<vendor>/<name>` | AI agent work (Claude, Cursor, Gemini) |

Sub-trees under `~/airepos/claude/`:
- `code/` — standalone agent projects
- `cowork/` — human + agent collaboration

### Conventions

- **Default**: human clones go to `~/repos/`, agent clones go to `~/airepos/`
- **Not enforced**: `--tree` override allowed when the default doesn't fit
- **Auto-detect**: if `CLAUDE_AGENT` env var is set, default to `~/airepos/claude/code/`
- **Special repos**: `~/dotfiles` and `~/dotclaude` live at home root (managed by chezmoi directly)
- **Scaffold guarantee**: directory structure exists before any clone (created by `run_once_after_10-scaffold-directories.sh.tmpl`)

### Machine-specific paths

Personal machines (arechste-mini, tutnix) get the full tree. Work machines omit `~/repos/gitlab.com/` and vendor-specific airepos directories. Controlled by chezmoi conditionals (`{{ if not .isWork }}`).

## Clone Lifecycle

### `ws clone`

1. Parse input: URL, `owner/repo`, or bare name (assumes `github.com/arechste/`)
2. Determine target tree based on convention (repos or airepos)
3. Select clone strategy (see below)
4. Clone via `gh repo clone` (handles auth, protocol, fork upstream automatically)
5. Register in local clone registry

#### Clone Strategies

| Strategy | Flag | Command suffix | When to use |
|----------|------|----------------|-------------|
| Full | (default) | — | Repos under ~1GB, standard development |
| Blobless | `--blobless` | `-- --filter=blob:none` | Large repos, ongoing development (blobs fetched on checkout) |
| Sparse | `--sparse <dir>` | `-- --filter=blob:none --sparse` + `git sparse-checkout set <dir>` | Monorepos, work in specific subdirectories |
| Shallow | `--shallow` | `-- --depth=1` | Inspection-only, throwaway clones |
| Mirror | `--mirror` | `git clone --mirror` | Backup/archival automation |

Notes:
- Blobless is the recommended strategy for repos over ~1GB; all git operations work except offline use
- Shallow clones degrade on subsequent fetches (17x server cost) — never fetch from a shallow clone
- `--filter` flags are incompatible with `--depth`/`--shallow-since`
- `gh repo clone` auto-selects protocol from `gh config get git_protocol` (ssh or https)
- Forks automatically get an `upstream` remote; use `--no-upstream` to skip

### `ws remove`

Safe removal protocol:

1. **Verify clean state**: `git status`, `git stash list`, `git log origin/<default>..HEAD`
2. **Block if unsafe**: unpushed commits, stashed work, or dirty working tree (unless `--force`)
3. **Detect sensitive gitignored files**:
   - Scan `.gitignore` patterns for files with content
   - Skip build artifacts: `node_modules/`, `__pycache__/`, `.build/`, `dist/`, `target/`
   - Flag sensitive patterns: `.env*`, `*.pem`, `*.key`, `credentials.*`, `*.age`, `*.enc.*`
4. **Vault if needed**: encrypt sensitive files with SOPS+age, store in vault
5. **Clean up branches**:
   - Delete merged local branches: `git branch --merged main | grep -v '^\*\|main' | xargs git branch -d`
   - Prune stale remote-tracking refs: `git fetch --prune`
   - Report any unmerged branches (require `--force` to proceed)
6. **Remove clone directory**
7. **Update local registry**

### `ws list` / `ws status`

- List all registered clones with optional git status
- Per-clone: branch, dirty file count, ahead/behind remote
- `--json` output for tooling integration

### `ws sync`

- Fetch and pull repos on default branch
- Repos on feature branches: report only, don't auto-pull
- Optional `--category` filter using repo-inventory.json

### `ws switch`

- Shell function (not subprocess — needs to `cd`)
- Tab completion from clone registry

## Vault

Encrypted backup of sensitive gitignored data before clone removal.

### Storage

```
~/.local/share/workspace-vault/<owner>/<repo>/
  manifest.json          # what was vaulted, when, original path
  secrets.enc.tar.gz     # SOPS+age encrypted archive
```

### Encryption

- Tool: SOPS with age backend
- Key: `~/.config/sops/age/keys.txt` (managed by dotfiles)
- Master keys / PATs: retrieved via 1Password CLI (`op`) with service account or OAuth

### Operations

- `ws vault list` — show vaulted repos
- `ws vault restore <repo>` — decrypt and copy back after re-clone

## Fleet distribution

Two repos are **chezmoi-distributed** — their working tree feeds files that chezmoi deploys to user-facing locations:

- `dotclaude` → deploys to `~/.claude/` (skills, rules, hooks)
- `dotfiles` → deploys to `~/.config/`, `~/.zshrc`, etc.

A merged PR in either repo only updates the source. Each consuming machine must run `chezmoi apply` for the change to be in effect. See `docs/conventions/fleet-sync.md` for the full loop and per-machine roles.

Other repos (`git-organizer`, `mac-organizer`) are not chezmoi-distributed — pulling main is enough.

## Multi-Machine

Both arechste-mini and tutnix use the same `ws` tooling (deployed via chezmoi). Clones are independent per machine — no automatic sync between them.

### Compare (future)

- `ws compare <machine>` — SSH to target, run `ws list --json`, diff against local
- Report: cloned here but not there, version differences

### Deploy (future)

- `ws deploy <repo> <machine>` — SSH clone a repo on a remote machine
- Requires: SSH keys, shared `ws` tooling (guaranteed by chezmoi)

## Local Clone Registry

File: `~/.local/share/workspace/clones.json`

```json
{
  "machine": "arechste-mini",
  "clones": [
    {
      "repo": "arechste/git-organizer",
      "path": "~/repos/github.com/arechste/git-organizer",
      "tree": "repos",
      "clonedAt": "2026-03-01T10:00:00Z",
      "lastSeen": "2026-03-14T08:00:00Z"
    }
  ]
}
```

### Integration with repo-inventory.json

The `localClones` field in repo-inventory.json aggregates clone data across machines:

```json
{
  "name": "git-organizer",
  "localClones": [
    { "machine": "arechste-mini", "path": "~/repos/github.com/arechste/git-organizer", "lastSeen": "2026-03-14T08:00:00Z" },
    { "machine": "tutnix", "path": "~/repos/github.com/arechste/git-organizer", "lastSeen": "2026-03-12T15:00:00Z" }
  ]
}
```

Populated by `tools/workspace/update-inventory.sh`, not by `fetch-repos.sh`.

## Worktree Lifecycle

Git worktrees provide isolated working directories sharing a single `.git` store. Preferred over stashing for parallel work, agent isolation, and hotfixes.

### Creation

```bash
# Manual: create worktree with new branch
git worktree add -b feat/thing .claude/worktrees/thing main

# Claude Code: automatic worktree for agent isolation
claude --worktree my-task        # creates .claude/worktrees/my-task/
```

- Same branch cannot be checked out in two worktrees simultaneously
- `worktree.guessRemote` (default true) auto-matches remote-tracking branches
- Add `.claude/worktrees/` to `.gitignore`

### Use

- Each worktree has its own HEAD, index, and working tree
- Hooks, config, and objects are shared with the main worktree
- Dependencies (node_modules, venvs) are NOT shared — must reinstall per worktree
- Use `extensions.worktreeConfig` for per-worktree git config

### Cleanup (order matters)

After PR merge, clean up in this exact order:

```bash
# 1. Remove the worktree (refuses if dirty; use --force to override)
git worktree remove .claude/worktrees/thing

# 2. Delete the local branch (safe: checks merge status)
git branch -d feat/thing

# 3. Prune stale remote-tracking refs
git fetch --prune
```

- Never `rm -rf` a worktree directory — always use `git worktree remove`
- A branch checked out in any worktree cannot be deleted until that worktree is removed
- `git worktree prune -n -v` shows stale entries without acting (dry-run)
- `git gc` auto-prunes worktrees older than 3 months (`gc.worktreePruneExpire`)
- Locked worktrees (`git worktree lock`) survive automatic pruning

### Claude Code Integration

- `isolation: worktree` in agent frontmatter gives each subagent its own worktree
- No-change worktrees are removed automatically; changed worktrees prompt user
- `git worktree list --porcelain` provides machine-readable output for scripting

### Multi-Session Safety

**Problem:** Two Claude sessions (or a Claude session + a human) running in the same working tree share one `HEAD`. A `git switch`, `git pull`, or `git checkout` in either session silently flips HEAD for everyone. Uncommitted changes appear to "follow" the switch because git keeps them in the index — the confusion is only discovered at the next `git status`. This is what happened during PR #416 (#411): a parallel session flipped HEAD back to `main` mid-task; recovery was clean only because the user spotted it before staging.

**Resolution: one worktree per parallel session (mandatory standard as of #417).**

#### Before opening a second session

```bash
# Option A (preferred): create a worktree for the new work
git worktree add -b feat/N-slug .claude/worktrees/slug main
# Then open the second session with its working directory set to .claude/worktrees/slug/

# Option B (fallback): use a separate clone
gh repo clone arechste/git-organizer ~/airepos/claude/cowork/git-organizer-2
```

Never open a second Claude session pointing at the same working tree without first isolating it into a worktree.

#### Detecting peer sessions

```bash
# Count active worktrees (1 = just main tree, >1 = peer worktrees exist)
git worktree list --porcelain | grep -c '^worktree'

# List all worktrees with their branch
git worktree list
```

If count > 1, confirm no other session is live before switching branches in the main tree.

#### Advisory lock (soft enforcement)

The dotclaude `PreToolUse` hook writes `.git/.claude-session-lock` (PID + branch + timestamp) when a session starts a `git switch` or `git pull`. On the next `git switch` / `git pull` / `git checkout`, the hook:

1. Checks for a stale or peer lock
2. Warns if a different branch/session is recorded
3. Does **not** hard-block (the worktree standard is the primary defense)

Lock file format:
```
pid=<PID>
branch=<current-branch>
started=<ISO-timestamp>
```

Stale lock detection: if `kill -0 <PID>` exits non-zero, the lock is orphaned and safe to remove.

#### What NOT to do

- Do not tell a user that parallel sessions in the same working tree are safe — they share HEAD
- Do not assure that uncommitted changes won't be affected by a peer session's `git switch`
- Do not skip worktree setup to "save time" — the recovery cost is higher than the setup cost

## Tooling Ownership

| Component | Repo | Deployed via |
|-----------|------|-------------|
| Convention doc | git-organizer | This file |
| Schema (localClones) | git-organizer | repo-inventory.schema.json |
| `ws` scripts | dotfiles | chezmoi → `~/.local/bin/ws` |
| Shell integration | dotfiles | chezmoi → `dot_zsh/functions.zsh` |
| Inventory sync | git-organizer | `tools/workspace/update-inventory.sh` |
