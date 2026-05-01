# Secrets Audit

Audit checklist for verifying that repos audited by git-organizer follow the canonical secret-handling conventions. git-organizer does NOT define secret-handling tooling — cross-project secret conventions live in `arechste/dotfiles`:

- **ADR-007** — canonical credential stack (op CLI + 1Password SSH agent + gh CLI + chezmoi templates)
- **ADR-008** — bootstrap auth sequence
- **ADR-009** — seal/unseal session-broker pattern (op signin TTL model)
- **ADR-010** — per-project secrets convention (.op.env + op run --env-file)

## Rule

When a repo audited by git-organizer needs to handle secrets, validate that:

1. The repo follows `arechste/dotfiles` ADR-010 (`.op.env` + `op run --env-file`)
2. OR the repo is one of the named exceptions in ADR-010:
   - **Exception 1**: files-encrypted-in-git via sops + age (e.g., mac-organizer inventory, ale-home-ops k8s manifests)
   - **Exception 2**: headless / CI contexts using OP_SERVICE_ACCOUNT_TOKEN
3. The repo does NOT roll its own credential helper, secret broker, or vault abstraction.

If the repo deviates, file a remediation issue in that repo with a link to ADR-010.

## Cross-references

- arechste/dotfiles `docs/decisions/ADR-007-credential-management-ownership.md`
- arechste/dotfiles `docs/decisions/ADR-010-per-project-secrets.md`
