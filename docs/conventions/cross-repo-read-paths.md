# Cross-Repo Read Paths

Which filesystem paths a Claude session in `git-organizer` can reliably read, and how that access is granted.

## Source of truth

Read access is controlled at two independent layers:

1. **Claude permission matcher** (`Read`, `Glob`, `Grep` tool grants) — governed by `settings.json` chain
2. **OS filesystem permissions** — governed by macOS user ownership (Claude runs as the logged-in user)

Both must allow the read for it to succeed.

## Claude-layer grants (settings chain)

The settings chain is loaded in this order; project `permissions.allow` **shadows** global (no union — see `permission-matrix.md`):

| Layer | File | Relevant grants |
|-------|------|-----------------|
| Global | `~/.claude/settings.json` | `Read(*)`, `Glob(*)`, `Grep(*)` |
| Project | `.claude/settings.json` | `Read(*)`, `Glob(*)`, `Grep(*)` |
| Local | `.claude/settings.local.json` | (does not further restrict reads) |

Because both layers grant `Read(*)`, the Claude permission matcher imposes **no path restriction on reads**. Any read that succeeds or fails does so at the OS layer.

## OS-layer reachable paths

Running as user `arechste` on macOS, everything under `$HOME` with default permissions is readable. The paths relevant to cross-repo work from `git-organizer`:

| Path | Purpose | Reliable? |
|------|---------|-----------|
| `~/airepos/claude/code/git-organizer/` | this repo (cwd) | Yes |
| `~/airepos/claude/code/*` | sibling Claude project repos (mac-organizer, dev-env, hookhub, …) | Yes |
| `~/airepos/{cursor,gemini,PAI,n8n-docker}/` | other AI-tool repos | Yes |
| `~/dotclaude/` | global Claude config **source** (chezmoi-managed) | Yes |
| `~/.claude/` | global Claude config **deployed target** (chezmoi output) | Yes (read-only discipline — edit source in `~/dotclaude/`) |
| `~/dotfiles/` | chezmoi dotfiles source | Yes |
| `~/repos/` | other personal repos | Yes |
| Anything outside `$HOME` | system paths | Not assumed — ask before reading |

## Paths NOT assumed readable

- Other user home directories (`/Users/<other>/`)
- System-restricted paths (`/private/var/db/`, `/Library/Keychains/`, etc.)
- Secrets on disk: `*.pem`, `*-key.json`, `.env` files (readable by OS, but **never read** by Claude — see `rules/secrets.md`)
- Anything under `sops`-encrypted or `age`-encrypted at rest (don't decrypt)

## When to use this doc

- A cross-repo read appears to fail — check OS path first, then settings chain
- Planning work that reads from a sibling repo — confirm it's in the table above
- Adding a new cross-repo workflow — if the source path isn't listed, document it here first

## Related

- `rules/permission-tiers.md` — autonomy levels (read/write boundaries)
- `docs/conventions/permission-matrix.md` — action matrix, shadowing rules
- `rules/secrets.md` — what never to read regardless of grants
