# Adopting git-organizer in a Consuming Repo

How another repo (yours or someone else's) consumes git-organizer's authored surfaces (skills, agents, conventions, references, hooks).

## Two consumption modes

### Mode 1 — No-clone (lazy, online)

Add this single line to the consuming repo's `CLAUDE.md`:

```markdown
@https://raw.githubusercontent.com/arechste/git-organizer/main/data/forge-index.json — git-organizer surfaces (skills, agents, conventions, references, hooks)
```

What you get:
- Lazy-loaded URLs in the index — Claude Code fetches an entry only when it's needed
- No git clone, no `additionalDirectories`
- Cached at the CDN layer (`raw.githubusercontent.com`), no GitHub API quota

What you don't get:
- Offline access (CI containers, fresh machines)
- File-modification access via Edit/Write — `additionalDirectories` mode is required for that

### Mode 2 — Clone + `additionalDirectories` (offline, eager)

For repos that need offline access or want to edit git-organizer-owned files in-session:

1. Clone git-organizer to the canonical filesystem path:
   ```sh
   git clone git@github.com:arechste/git-organizer.git ~/airepos/claude/code/git-organizer
   ```
2. In the consuming repo's `.claude/settings.json`:
   ```json
   {
     "additionalDirectories": ["~/airepos/claude/code/git-organizer"]
   }
   ```
3. Claude Code resolves the path and loads `.claude/rules/` (and other declared surfaces) automatically.

`additionalDirectories` is the same pattern documented in `inheritance-model.md`. The two modes can coexist — Mode 1 for lazy reference, Mode 2 for eager file operations.

## Versioning

Each entry in the index carries a `version` field (semver). Consuming repos may pin to a specific commit-SHA URL when stability matters:

```
https://raw.githubusercontent.com/arechste/git-organizer/<sha>/.claude/skills/repo-audit/SKILL.md
```

The `main`-pinned URL (default in the index) tracks living references. SHA-pinned URLs are appropriate for templates, action workflows, or anywhere reproducibility outweighs freshness.

## Deprecation

When an entry's `status` flips to `deprecated`, the entry's `replacedBy` field names the successor. git-organizer guarantees ≥1 minor of overlap before removal. Consumers should migrate during the overlap window.

## Anti-pattern: copying

Do **not** copy git-organizer's files into the consuming repo. Copies diverge silently (`inheritance-model.md` § Anti-pattern). The detection sentinel is the header `# Source: git-organizer/...` — any file outside git-organizer that carries it is a stale copy to remediate.

## Bootstrap fallback

Disconnected environments (CI containers, fresh machines, new contributors):

- **Fresh machine**: clone git-organizer as part of standard workspace setup (already documented in dotfiles `airepos/` layout)
- **CI containers**: rules from `additionalDirectories` aren't loaded in CI — Claude Code sessions don't run there. Use the deterministic shell scripts in `tools/` instead.
- **New repo bootstrap**: a one-time rule snapshot in `.claude/rules/` can ship with the repo, then transition to `additionalDirectories` once git-organizer is available locally. Document the transition in the repo's CLAUDE.md.

## Refs

- `data/forge-index.json` — the index
- `data/schemas/forge-index.schema.json` — entry shape
- `.claude/rules/inheritance-model.md` — ownership and rule layering
- #396 — adoption contract (D2 transport, D5 hooks ownership)
- #411 — Pillar 2 (Author) implementation tracking
