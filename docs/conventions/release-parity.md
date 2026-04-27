# Release Parity

A release is **three artifacts** that must match exactly:

1. **Git tag** — `vX.Y.Z` annotated tag pushed to the default branch
2. **GitHub release** — `gh release view vX.Y.Z` returns a published release with non-empty notes
3. **Changelog entry** — `## vX.Y.Z` section in `CHANGELOG.md` on the default branch

If any of the three is missing or out of sync, the release is **broken** even if other downstream consumers don't notice yet. This convention captures the assertion as a checkable property.

## Why this matters

Each artifact serves a different consumer:

| Artifact | Consumer | Failure mode if missing |
|----------|----------|-------------------------|
| Git tag | Git users, dependency pinning | Cannot pin to version; `git describe` reports older tag |
| GitHub release | Web viewers, release-watchers (`gh release list`), automation | Release page 404; notification systems don't fire |
| Changelog entry | Humans reading the repo, downstream changelog aggregators | Repo looks abandoned at this version; aggregators skip it |

Drift here is the silent kind — the merge commit and tag exist, but the release page is empty or the changelog skipped a version. dotfiles' historical "9 missing releases" incident (`/dc:ship` step 8 references it) is the canonical example.

## The assertion

For every released version vX.Y.Z, all three must hold:

```bash
# 1. Tag exists, points to a commit on main
git rev-parse vX.Y.Z >/dev/null

# 2. GitHub release exists with non-empty notes
gh release view vX.Y.Z --json tagName,body --jq '.tagName == "vX.Y.Z" and (.body | length) > 0'

# 3. CHANGELOG.md has a matching section
grep -F "## vX.Y.Z" CHANGELOG.md
```

A repo passes the parity check iff every git tag has corresponding entries in 2 and 3.

## Where this is enforced

| Surface | Behavior |
|---------|----------|
| `/dc:ship` | Final report lists the three artifacts and their parity state (per `docs/conventions/change-verification.md` "Verification done?" line). Step 8 (GitHub release) and CHANGELOG generation are part of the skill flow already. |
| `release.yml` (CI) | Triggers on tag push, creates the GitHub release from `CHANGELOG.md`. The CI run IS the parity assertion for steps 2 ↔ 3. If CI fails, the GitHub release is missing. |
| `docs/conventions/semver.md` | Release Checklist references this doc — the checklist's last items (8: GitHub release, 9: tag push) are the parity-creating steps. |
| Audit (stretch) | A dedicated CI job that runs the assertion across all tags would surface drift retroactively. Not implemented yet; tracked as an optional follow-up. |

## What this convention is not

- Not a substitute for `release.yml`. The CI workflow does the work; this doc names the property the workflow is enforcing.
- Not a CI-blocking gate on every PR. The check runs at release time only.
- Not the same as "the tag exists." A tag without a release page or changelog entry is a parity failure.

## References

- Audit: `docs/audit/2026-04-18-git-workflow-audit.md` § Inconsistency #5
- Related: `docs/conventions/semver.md` (Release Checklist)
- Related: `docs/conventions/change-verification.md` (release-parity is the bar `/dc:ship` reports)
- Implementation: `~/dotclaude/home/skills/ship/SKILL.md` step 8 (GitHub release CI-driven, with fallback)
