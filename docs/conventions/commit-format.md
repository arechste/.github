# Conventional Commit Format

All commits across managed repos follow the conventional commit specification.

## Format

```
type(scope): description

[optional body]

[optional footer]
```

## Types

| Type | When to use |
|------|-------------|
| `feat` | New feature or capability |
| `fix` | Bug fix |
| `refactor` | Code change that neither fixes nor adds |
| `docs` | Documentation only |
| `test` | Adding or fixing tests |
| `chore` | Maintenance, deps, config |
| `ci` | CI/CD pipeline changes |
| `perf` | Performance improvement |
| `style` | Formatting, whitespace (no logic change) |
| `build` | Build system or tooling |
| `revert` | Reverting a previous commit |
| `wip` | Work in progress (will be squashed) |

## Rules

- Subject line under 72 characters
- Imperative mood: "add feature" not "added feature"
- Scope is optional but recommended: `feat(auth): add login`
- Breaking changes: append `!` before colon: `feat(api)!: change response format`
- Reference issues: `feat(labels): add emoji support (#5)` or `Fixes #5` in body
- AI attribution trailer: `Co-Authored-By: Claude/{model}/{identity}@{hostname} <noreply@anthropic.com>`
- One logical change per commit

## Trailer Anatomy

AI-assisted commits use a structured trailer for traceability:

```
Co-Authored-By: Claude/Opus/Auditor@arechste-mini <noreply@anthropic.com>
                 │     │     │        │
                 │     │     │        └─ hostname (hostname -s)
                 │     │     └────────── identity (role or repo name)
                 │     └──────────────── model family (Opus, Sonnet, Haiku)
                 └────────────────────── AI family (constant)
```

### Identity resolution (hybrid)

Named roles are used for key repos; all others fall back to the repo name:

| Repo | Identity | Source |
|------|----------|--------|
| git-organizer | `Auditor` | mapped role |
| dotclaude | `Builder` | mapped role |
| dotfiles | `Ops` | mapped role |
| (any other) | repo name | fallback |

Role mappings are stored in `~/.claude/commit-trailer.json` and consumed by the `dc:commit` skill and the `commit-msg` git hook.

### Enforcement

- **dc:commit skill**: resolves identity from config, generates correct trailer
- **commit-msg hook**: rewrites any generic `Co-Authored-By:.*Claude.*` trailer to the correct format
- **Config**: `~/.claude/commit-trailer.json` maps repo names to role identities

## Examples

```
feat(labels): add emoji support for label categories

fix: resolve null pointer when parsing empty inventory

docs(conventions): add commit format reference

chore(deps): update actions/checkout to v4.2.2

ci(workflows): add template drift detection to scheduled audit

refactor(audit): extract scoring logic to separate function

wip(maturity): partial implementation of scoring dimensions
```

## Squash Merge

We use squash merge exclusively. The PR title becomes the final commit message on main. Individual commits on feature branches are squashed away, so:

- PR title MUST follow conventional format
- Individual commits SHOULD follow format (warned but not enforced)
- `wip:` commits are fine on branches — they get squashed

## Enforcement

- **Local**: pre-commit hooks (via dotfiles) validate format
- **CI**: `convention-check.yml` validates PR titles
- **Skills**: `dc:commit` ensures proper format in Claude Code
