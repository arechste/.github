# Machine Labels

How the `machine/*` label family is resolved and consumed. Canonical data lives in `data/machine-groups.json` (schema: `data/schemas/machine-groups.schema.json`).

## Label kinds

Every `machine/*` label falls into one of three kinds:

| Kind | Example | Resolver output |
|------|---------|-----------------|
| Literal hostname | `machine/cr61790dfv` | `{hosts: [cr61790dfv], fanout: false}` |
| Alias | `machine/corp-laptop` | Recursively resolves to the target hostname |
| Quantified role | `machine/all-personal`, `machine/any-corp` | Expands role to hostnames; `all-*` sets `fanout: true`, `any-*` sets `fanout: false` |

Two legacy sentinels remain:

| Sentinel | Meaning today |
|----------|---------------|
| `machine/all` | Every host in `hosts` — `fanout: true` |
| `machine/any` | Every host in `hosts` — `fanout: false` |
| `machine/linux` | Placeholder. Currently unresolved (no Linux hosts registered). Resolver errors on it until `roles.linux` is populated. |

## Resolver algorithm

Input: a `machine/<x>` label (the `<x>` suffix).
Output: `{hosts: [string], fanout: bool}` or an error.

```
resolve(x):
  if x in hosts:                  return ([x], false)
  if x in aliases:                return resolve(aliases[x])   # recurse, with cycle guard
  if x == "all":                  return (all hosts, true)
  if x == "any":                  return (all hosts, false)
  if x starts with "all-":
    role = x[4:]
    if role in roles:             return (roles[role], true)
  if x starts with "any-":
    role = x[4:]
    if role in roles:             return (roles[role], false)
  error: unknown machine label
```

The implementation lives in `tools/lib/machine-resolver.sh`. Consumer scripts (this repo) and skills (via `additionalDirectories`) source that helper.

## Pickup rule

Every consumer that decides whether a given host can work on an issue applies the same rule:

```
eligible-on(host, label) = host ∈ resolve(label).hosts
```

If the issue has no `machine/*` label, it is eligible anywhere (same as `machine/any`).

If the issue has multiple `machine/*` labels (v1 does not create these, but sync drift might introduce them), eligibility is the union: any matching label makes the current host eligible. The `fanout` flag is the OR across labels.

## Fan-out rule

At triage, for any issue whose resolved `machine/*` label has `fanout == true` and `|hosts| > 1`:

1. Recommend `./tools/maintenance/fanout-issue.sh --issue N --apply`.
2. The tool creates one sub-issue per resolved host, each labeled with `machine/<hostname>` (literal).
3. The parent gets `meta/tracking` added. Per `docs/conventions/issue-hierarchy.md`, a `meta/tracking` parent stays open until manually closed; sub-issues drive the actual work.

Edge cases:

- `fanout == true` with 1 resolved host → no fan-out needed. The tool exits 0 with a "convert to literal `machine/<hostname>`" recommendation.
- `fanout == true` with 0 resolved hosts → tool warns (empty role). Happens today for `machine/linux` if it were reintroduced as an empty role.
- Unknown label → resolver errors; session-start hook and triage surface it as a machine-label drift warning.

## Adding a new host

1. Add the hostname to `data/machine-groups.json` under `hosts` with its `role`.
2. Add it to the appropriate `roles[<role>]` array.
3. Add the `machine/<hostname>` entry to `data/label-definitions.json` (infrastructure category machine block).
4. Run `./tools/labels/sync-labels.sh --dry-run git-organizer` then without `--dry-run` to land the new label.
5. Run the sync against each sibling repo (`dotfiles`, `mac-organizer`) that adopts the convention.

## Adding a new role

1. Add the role to `data/machine-groups.json` under `roles` with its member hostnames.
2. Add the two quantified labels to `data/label-definitions.json`:
   - `machine/all-<role>` and `machine/any-<role>`
3. Sync labels as above.

Do NOT add `machine/<role>` as an unquantified label — always use the `all-` / `any-` prefix to make fan-out intent explicit at the label level.

## Adding an alias

1. Add to `data/machine-groups.json` under `aliases` as `"<alias>": "<target>"`.
   `<target>` can be a hostname or another alias (resolved recursively).
2. Add the `machine/<alias>` entry to `data/label-definitions.json`.
3. Sync labels.

Aliases should only be used for single-host shortcuts (e.g., `corp-laptop` for `cr61790dfv`). For multi-host groups, use a role, not an alias.

## Failure mode: unknown label

The resolver fails closed — an unknown `machine/<x>` label yields an error, which consumers treat as "not eligible on this host." This prevents a typo from accidentally broadening pickup eligibility. The session-start hook and `/triage` surface unknown-label warnings so drift is visible.
