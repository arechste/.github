# Fleet Sync Loop

The full loop from local edit to "session-safe on every machine" for chezmoi-distributed repos. Session-protocol stops at `git pull` and is silent about the propagation step that actually makes a change effective.

## Distribution model

Two repos in this project are **chezmoi-distributed** — their working tree is the source for files deployed by chezmoi to user-facing locations:

| Repo | Source location | Deployed to | Apply command |
|------|-----------------|-------------|---------------|
| `dotclaude` | `~/dotclaude/home/` | `~/.claude/` (skills, rules, hooks, settings) | `chezmoi apply` |
| `dotfiles` | `~/dotfiles/` | `~/.config/`, `~/.zshrc`, etc. | `chezmoi apply` |

A merged PR in either repo does **not** take effect anywhere until `chezmoi apply` runs on each consuming machine.

Other repos (`git-organizer`, `mac-organizer`) are **not** chezmoi-distributed — a merged PR is in effect immediately on any machine that pulls main.

## End-to-end loop

```
┌──────────────┐   ┌─────┐   ┌───────┐   ┌──────────────┐   ┌────────────────┐
│ local change │──▶│ PR  │──▶│ merge │──▶│ chezmoi apply │──▶│ session-safe   │
│ (edit + test)│   │ open│   │       │   │ on machine N  │   │ on machine N   │
└──────────────┘   └─────┘   └───────┘   └──────────────┘   └────────────────┘
                                              │
                                              ├── arechste-mini  (primary)
                                              ├── tutnix
                                              └── (any other registered host)
```

The loop closes per-machine, not per-merge. A merge only puts the change in `~/dotclaude/` or `~/dotfiles/` (the source). Until `chezmoi apply` runs on a machine, that machine still sees the **previous** version of every deployed file.

## Roles

| Step | Owner | Action |
|------|-------|--------|
| 1. local change | author | edit the source under `~/dotclaude/` or `~/dotfiles/` |
| 2. PR open | author | `git push`, `gh pr create` |
| 3. merge | author or auto-merge | `gh pr merge --squash --delete-branch` |
| 4. chezmoi apply (primary) | author | runs `chezmoi apply` on the primary machine before considering the change "live" |
| 5. chezmoi apply (other machines) | session start on each machine | the next session-start hook on machine N picks up the merged commit; author files a tracking note if a non-self machine needs explicit pickup |
| 6. session-safe | session start | the deployed file is what skills/rules/hooks load |

The author is responsible for **step 4**. Steps 5 and 6 happen at the next session on each consuming machine — typically zero ceremony.

## Session-end behavior

Per `.claude/rules/session-protocol.md` Session end, when the merged change is in a chezmoi-distributed repo:

1. Note in the report which deployed paths the change touches (e.g., "deploys to `~/.claude/skills/dc/work/SKILL.md`").
2. Run `chezmoi apply` on the primary machine (or note "applied" if the session itself triggered it).
3. If other machines need the change urgently (e.g., a fix to a session-start hook that's failing on tutnix), file a tracking note in the merged PR or open a follow-up issue.

The default is **passive propagation** — the next session on each machine picks up the change. Active propagation (SSH-driven `chezmoi apply` on remotes) is out of scope by policy (see `~/.claude/CLAUDE.md` § "Local only by default").

## Session-start coverage

Session-start hooks already track machine-divergence:

- `dotclaude`: `session-start-deferred-universal.sh` flags hostname mismatches against `machine/*` labels
- `dotfiles`: ADR-008 bootstrap auth sequence ensures secrets are available before any session work

These hooks **do not** run `chezmoi apply` automatically — that decision belongs to the user. They only surface the drift.

## Verification

This convention's verification bar (per `docs/conventions/change-verification.md`).

### Non-distributed repos (git-organizer, mac-organizer)

A merged PR is in effect on any machine that pulls. Verification is a single command:

```bash
git pull
```

Expected output confirms fast-forward to the merge commit:

```
From github.com:arechste/<repo>
   abc1234..def5678  main -> origin/main
Updating abc1234..def5678
Fast-forward
 <changed file>  | <N> +/-
```

### Chezmoi-distributed repos (dotclaude, dotfiles)

Three steps per machine. Run these on **each** consuming machine (arechste-mini, tutnix).

**Step 1 — Pull the source repo**

```bash
git -C ~/dotclaude pull   # or ~/dotfiles for dotfiles changes
```

Expected: fast-forward to the merge commit, no conflicts.

**Step 2 — Dry-run to preview what chezmoi will write**

```bash
chezmoi apply --dry-run --verbose
```

Expected: a list of file paths that would be updated with `modify` or `create` verbs. No error output. Example:

```
modify    /Users/arechste/.claude/skills/dc/work/SKILL.md
```

If `chezmoi apply --dry-run` exits non-zero or prints conflict lines (`conflict`, `error`), do **not** proceed to step 3 — investigate the source diff first.

**Step 3 — Apply**

```bash
chezmoi apply
```

Expected: silent exit 0 (chezmoi only prints when something changes or errors). Verify the deployed file matches the source:

```bash
chezmoi diff   # should show no output after a clean apply
```

### Automated counterpart

`tools/audit/main-cleanliness.sh` (#316) automates steps 1–2 for CI contexts: clones `origin/main` into a temp dir and runs `chezmoi apply --dry-run` for chezmoi-distributed repos, reporting any conflicts. Use it to verify the invariant without modifying the live deployment.

## What this convention is not

- Not an SSH-orchestrated rollout. The fleet is sync'd lazily, machine by machine, on the next session.
- Not a replacement for `merge-completion.md`. That doc covers `--delete-branch` cleanup; this doc covers what happens after the merge lands.
- Not a CI gate. Like `change-verification.md`, this is guidance, not enforcement.

## References

- Audit: `docs/audit/2026-04-18-git-workflow-audit.md` § Axis 2, Inconsistency #6
- Related: `docs/conventions/workspace-management.md` § Fleet distribution
- Related: `docs/conventions/change-verification.md` (chezmoi-distributed bar)
- Related: `.claude/rules/session-protocol.md` (Session end references this doc)
