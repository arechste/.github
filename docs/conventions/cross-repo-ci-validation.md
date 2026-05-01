# Cross-Repo CI Validation

Pattern for validating one repo's content against another repo's source-of-truth files from within a GitHub Actions workflow.

Reference implementation: `arechste/dotfiles#680` — validates the `CLAUDE.md` machine table against `mac-organizer/inventory/*.yaml`.

## When to Use This Pattern

Use cross-repo CI validation when:

- A file in repo A contains data **derived from** or **dependent on** repo B's source of truth
- The validating repo does **not** own the source data
- You need drift detection without centralizing logic into git-organizer

**Examples in scope:**

| Validating repo | Source repo | Validates |
|-----------------|-------------|-----------|
| dotfiles | mac-organizer | Machine table in `CLAUDE.md` vs `inventory/*.yaml` |
| dotfiles | git-organizer | Label/convention references |
| dotfiles | dotclaude | Skill/hook references |

**Not for:**

- Write actions from CI (separate pattern — higher risk, needs separate treatment)
- Content the validating repo owns
- Files requiring SOPS decryption (CI can't access age keys; use plaintext top-level fields only)
- Cross-org reads to `ntnxlab-ch` or self-hosted forge (not in scope until explicitly planned)

---

## Secret Naming Convention

### Name

`GH_CROSS_REPO_TOKEN` — singular, generic. One PAT covers multiple read targets.

Rationale: a per-target name like `DOTFILES_READ_TOKEN` would proliferate secrets as cross-repo validations are added. A single generic token with `Contents: Read` across owned repos is sufficient for this read-only pattern.

### 1Password Storage

- **Vault:** `ALE-AUTOMATION` (automation tier per `arechste/dotfiles docs/security/access-matrix.md`)
- **Item name:** `GH_CROSS_REPO_TOKEN`
- **Required metadata fields on the 1P record:**
  - `target_repos` — comma-separated list of repos the token covers (e.g., `arechste/mac-organizer, arechste/git-organizer`)
  - `scopes` — human-readable scope summary (e.g., `Contents: Read-only, Metadata: Read-only`)
  - `created` — ISO 8601 date
  - `expires` — ISO 8601 date
  - `rotation_reminder` — calendar reminder set? (yes/no)
  - `github_secrets` — list of `repo/SECRET_NAME` entries where a copy is stored

Update `target_repos` whenever the token is shared with an additional workflow.

---

## Token Shape

| Property | Value |
|----------|-------|
| Type | Fine-grained PAT (preferred over classic) |
| Resource owner | `arechste` (user that owns target repos) |
| Repository access | Selected repos — only what's needed |
| Permissions | `Contents: Read-only`, `Metadata: Read-only` (auto) |
| Expiration | 1 year max |
| Rotation | Update 1P record → regenerate on GitHub → update repo secret |

Fine-grained PATs are preferred because:
- Scopes are narrower and auditable per-permission
- Repository access can be limited to specific repos (not "all repositories")
- GitHub generates clearer audit log entries

### Future: GitHub App

When managing multiple orgs (e.g., `ntnxlab-ch`), migrate to a GitHub App:
- Auto-rotating tokens (no manual rotation)
- Per-repo installation with granular permissions
- Audit trail for all operations
- No personal account dependency

Migration path: replace `GH_CROSS_REPO_TOKEN` with an App installation token in workflows. Read scripts don't change — they use `GH_TOKEN` env var regardless.

---

## CI Workflow Shape

### Auth

```yaml
- name: Read from target repo
  env:
    GH_TOKEN: ${{ secrets.GH_CROSS_REPO_TOKEN }}
  run: |
    gh api repos/arechste/mac-organizer/contents/inventory/hosts.yaml \
      --cache 24h \
      --jq '.content' | base64 -d
```

### Read pattern

```bash
gh api repos/<owner>/<repo>/contents/<path> --cache 24h --jq '.content' | base64 -d
```

- `--cache 24h` is appropriate for inventory/convention files (change cadence is daily at most)
- Use `--jq '.content'` then `base64 -d` — GitHub API returns file content as base64
- Read only **plaintext top-level fields** — never trigger SOPS decryption from CI
- For YAML files with encrypted subtrees (sops + age): read fields that are not encrypted

### Graceful skip

```yaml
- name: Validate machine table
  env:
    GH_CROSS_REPO_TOKEN: ${{ secrets.GH_CROSS_REPO_TOKEN }}
  run: ./scripts/validate-machine-table.sh
```

```bash
# In the script:
if [[ -z "${GH_CROSS_REPO_TOKEN:-}" ]]; then
  echo "::warning::GH_CROSS_REPO_TOKEN not set — skipping cross-repo validation"
  exit 0
fi
```

Rules:
- Missing secret → warning annotation + `exit 0` (never block PRs on unconfigured CI)
- Failed validation → `exit 1` (check fails, PR is blocked — this is intentional)
- Network error → `exit 1` with a clear message (transient; re-run resolves)

### Schedule

- **Nightly cron** + **on PRs touching the file being validated**
- Cron catches upstream drift (target repo changed without triggering the validating repo's CI)
- PR trigger catches local drift (validating repo's file was edited)

```yaml
on:
  push:
    paths:
      - 'CLAUDE.md'         # file being validated
  pull_request:
    paths:
      - 'CLAUDE.md'
  schedule:
    - cron: '0 3 * * *'    # nightly at 03:00 UTC
```

---

## Adoption Checklist

When adding a new cross-repo validation workflow:

- [ ] Confirm the target repo is in scope (owned by `arechste`, plaintext fields)
- [ ] Verify `GH_CROSS_REPO_TOKEN` in `ALE-AUTOMATION` vault covers the new target repo; if not, regenerate with expanded access and update the 1P record's `target_repos` field
- [ ] Update the `github_secrets` field in the 1P record to note the new repo secret location
- [ ] Write the validation script with graceful skip and exit codes per this doc
- [ ] Set `--cache 24h` (or shorter if the source file changes more frequently)
- [ ] Add both the cron schedule and the PR path trigger
- [ ] Reference this doc in the workflow's comment header

---

## See Also

- `docs/conventions/cross-repo-automation.md` — write-side cross-repo operations (label sync, template sync)
- `docs/conventions/secrets-audit.md` — how git-organizer audits secret handling in repos
- `arechste/dotfiles docs/security/access-matrix.md` — auth tier model (ALE-AUTOMATION vault)
- `arechste/dotfiles#680` — reference implementation (machine table validation)
