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

## Detecting a Bump from Commits (automated tools)

Tools that compute a release-gap signal use the following rules to recommend a semver bump. The canonical implementation is `dotclaude/tools/release-gap.sh`.

### Input

```
git log <latest_tag>..HEAD --pretty=format:'%s'
```

Where `<latest_tag>` is the most recent `v*` annotated or lightweight tag reachable from HEAD (`git describe --tags --abbrev=0 --match 'v*'`).

- If no `v*` tag exists → no recommendation (repo not yet versioned).
- If no commits since the tag → no recommendation (nothing to release).

### Commit classification

Each commit subject is matched against the conventional-commit prefix pattern:

```
^<type>(<scope>)?!?:
```

Recognised types: `feat`, `fix`, `chore`, `docs`, `refactor`, `ci`, `test`.  
Subjects that don't match any recognised type are counted as `other` (no bump weight).

### Breaking-change detection

A commit is **breaking** if either:

- Its subject contains `!` before the colon: `feat!:` or `feat(scope)!:`
- Its full body contains the string `BREAKING CHANGE`

### Capability-adding chore (`capability_chore`)

A `chore(scope)` commit triggers a **minor** bump (not patch) when both conditions hold:

1. The scope contains one of: `tools`, `skills`, `templates`, `packages`
2. The commit added new files under those directories (detected via `git diff --diff-filter=A`)

Rationale: adding new tooling, skills, or templates expands capability even though the change is framed as `chore`.

### Bump recommendation

| Condition | Recommended bump |
|-----------|-----------------|
| Any breaking change detected | `major` |
| `feat:` count > 0 OR `capability_chore` is true | `minor` |
| Otherwise (only `fix`, `chore`, `docs`, `refactor`, `ci`, `test`) | `patch` |

This mirrors the human decision guide above — the automated rule is its machine-readable form.

### Adopting the signal

Any repo governed by git-organizer can adopt this signal:

1. Copy `dotclaude/tools/release-gap.sh` into the repo's `tools/` directory.
2. Wire it into your triage skill or session-start hook:  
   `tools/release-gap.sh --human` for inline output, `--json` for machine consumption.
3. No other dependencies — requires only `bash`, `git`, and `jq`.

The canonical implementation lives in dotclaude and is updated there. Downstream copies should track it via the `forge-index.json` adoption check or periodic drift review.

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

After tagging, verify **release parity** — git tag, GitHub release, and CHANGELOG entry must all reference the same version. See `docs/conventions/release-parity.md` for the assertion and where it's enforced.

## For Governed Repos

Repos don't need to follow this exactly. The minimum is:
- Use `vMAJOR.MINOR.PATCH` format
- Bump major for breaking changes
- Tag releases on the default branch
- Don't release on every commit — meaningful groupings only
