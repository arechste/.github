# Semantic Versioning Convention

Standard versioning rules for repos governed by git-organizer.

## Format

```
vMAJOR.MINOR.PATCH[-prerelease]
```

Examples: `v1.0.0`, `v2.3.1`, `v1.4.0-alpha.1`, `v1.4.0-rc.1`

## When to Bump

| Bump | When | Examples |
|------|------|---------|
| **Major** | Breaking changes that require consumers to adapt | API removed, schema field renamed, CLI interface changed, convention semantics altered |
| **Minor** | New capabilities that are backwards-compatible | New feature, new command, new convention doc, new integration |
| **Patch** | Bug fixes and corrections that don't add capabilities | Fix a bug, correct a typo, update dependencies, refresh data |

### Decision guide

Ask yourself:
1. **Will downstream consumers break?** → Major
2. **Does this add something new?** → Minor
3. **Does this fix something existing?** → Patch
4. **Is this only housekeeping (data sync, tracking)?** → Don't release yet, batch into the next minor or patch

## Pre-release Tags

Use pre-release suffixes when a milestone is in progress:

| Suffix | Meaning | When to use |
|--------|---------|-------------|
| `-alpha.N` | Work in progress | Features incomplete, milestone open |
| `-beta.N` | Feature complete | Stabilization, testing |
| `-rc.N` | Release candidate | Only critical fixes allowed |
| _(none)_ | Stable release | Milestone closed, all checks pass |

Promote to stable by tagging `vX.Y.Z` (no suffix) when the milestone closes.

## Release Cadence

- **Milestone-driven**: a milestone closing is the primary trigger for a minor release
- **Don't release every session**: accumulate changes, batch `chore(data):` commits
- **Hotfixes**: `vX.Y.Z+1` patch from a `hotfix/` branch for urgent fixes on current release
- **Pre-stable** (`0.x.y`): breaking changes allowed, minor bumps per milestone

## Release Checklist

1. All milestone issues closed
2. No open delegated issues blocking the milestone (`milestone-readiness.sh`)
3. `pre-commit run --all-files` passes
4. CHANGELOG.md updated
5. Version references updated (VERSION, package.json, pyproject.toml if applicable)
6. Release commit: `chore(release): vX.Y.Z`
7. Annotated tag: `git tag -a vX.Y.Z -m "description"`
8. Close milestone, create next version milestone
9. Push: `git push origin main vX.Y.Z`
10. GitHub release (optional): `gh release create vX.Y.Z --generate-notes`

## For Governed Repos

Repos don't need to follow this exactly. The minimum is:
- Use `vMAJOR.MINOR.PATCH` format
- Bump major for breaking changes
- Tag releases on the default branch
- Don't release on every commit — meaningful groupings only
