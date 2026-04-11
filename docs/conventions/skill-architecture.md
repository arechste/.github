# Skill Architecture Convention

How to structure Claude Code skills for context efficiency, DRY, and maintainability.

## Principles

1. **Lazy loading** — only skill name + description load at session start; full SKILL.md loads on-demand when triggered
2. **Progressive disclosure** — SKILL.md stays lean; detailed instructions live in `references/` and load only when needed
3. **Scripts as tools** — deterministic logic lives in `tools/`; script code never enters context, only output does
4. **Shared resources** — cross-skill references live in `shared/` to eliminate duplication
5. **Subagent preloading** — agent.md `skills:` field injects skill content into subagent context at startup

## Skill directory structure

```
skills/
├── shared/                          # Cross-skill references (DRY)
│   ├── label-taxonomy.md            # Shared domain knowledge
│   ├── repo-standards.md            # Shared conventions
│   └── git-workflow.md              # Shared workflow patterns
│
└── my-skill/
    ├── SKILL.md                     # Entry point (<500 lines, <2000 words)
    ├── references/                  # Detailed instructions (loaded on-demand)
    │   ├── advanced-patterns.md     # Referenced via markdown link from SKILL.md
    │   └── error-recovery.md
    ├── tools/                     # Deterministic executables (never in context)
    │   └── check.sh
    └── assets/                      # Templates, schemas, examples
        └── template.md
```

### What goes where

| Location | Content | Context impact |
|----------|---------|----------------|
| SKILL.md | Frontmatter + workflow steps + quick reference | Loaded when skill triggers |
| references/ | Detailed patterns, conversion tables, domain knowledge | Loaded on-demand via markdown link |
| tools/ | Bash/Python executables Claude runs via Bash() | Never in context (only output) |
| assets/ | Templates, schemas, boilerplate | Loaded when referenced |
| shared/ | Cross-skill references, shared domain knowledge | Loaded when any referencing skill needs it |

## SKILL.md guidelines

### Frontmatter

```yaml
---
name: dc:my-skill
description: "Concise description with trigger keywords. Use when the user asks to 'do X' or 'check Y'."
tags: [git, audit]
context: fork              # Optional: run in isolated subagent
agent: reviewer            # Optional: use custom agent type
allowed-tools:             # Restrict available tools
  - Read
  - Bash(./tools/*)
disable-model-invocation: true  # Optional: manual-only invocation
---
```

### Description rules

- Under 1024 characters
- Include concrete trigger phrases: "Use when the user asks to 'audit labels'"
- Third person, not "I can" or "You can"
- Specific keywords over vague categories
- This is the ONLY part that loads every session — make it count

### Body size targets

- **Target**: <500 lines, <2000 words
- **Maximum**: 5000 words before mandatory splitting
- **Split trigger**: if SKILL.md exceeds 300 lines, move detailed instructions to references/

### Referencing external files

```markdown
## Advanced patterns
For detailed label taxonomy, see [label-taxonomy.md](../shared/label-taxonomy.md).
For error recovery procedures, see [error-recovery.md](references/error-recovery.md).
```

Claude reads the referenced file only when it needs that information — not upfront.

## Tier model

Skills are organized into tiers that control auto-trigger behavior and context isolation.

| Tier | `disable-model-invocation` | `context: fork` | When to use |
|------|---------------------------|-----------------|-------------|
| **Core** | false (auto-trigger) | no | Interactive skills users invoke frequently (commit, review, test-run) |
| **Standard** | false (auto-trigger) | yes (fork) | Analysis/workflow skills that benefit from isolation (triage, release, audit) |
| **Specialist** | true (manual only) | varies | Utility skills rarely needed (stats, journal, skill management) |

### Tier assignment criteria

- **Core**: used in >50% of sessions, interactive (needs main context), fast
- **Standard**: used in 10-50% of sessions, analysis-heavy, can run isolated
- **Specialist**: used in <10% of sessions, admin/meta tasks

## Agent definitions with skill preloading

Agents can preload skills so subagents have domain knowledge available immediately.

```yaml
# agents/git-workspace.md
---
name: git-workspace
description: Handles git/GitHub workflows including commit, triage, release, review, CI, and repo health.
tools: Read, Write, Edit, Glob, Grep, Bash
skills:
  - dc:commit
  - dc:triage
  - dc:release
  - dc:repo-health
memory: project
---

Git/GitHub workspace agent. Follows conventions in shared/git-workflow.md.
```

### When to create an agent vs use context: fork

| Use case | Mechanism |
|----------|-----------|
| Skill needs isolated context | `context: fork` on the skill |
| Multiple skills needed together | Agent with `skills:` preloading |
| Custom tool restrictions | Agent with `tools:` field |
| Cross-session learning | Agent with `memory:` field |
| Read-only analysis | Agent with `tools: Read, Glob, Grep` (no write tools) |

## Hook gating

Hooks should gate on context to avoid unnecessary execution.

### Pattern: git-aware hooks

```bash
#!/usr/bin/env bash
set -euo pipefail

# Parse input
INPUT=$(cat)
CWD=$(echo "$INPUT" | jq -r '.cwd // empty')

# Gate: exit early if not in a git repo
if ! git -C "${CWD}" rev-parse --git-dir >/dev/null 2>&1; then
  exit 0
fi

# Git-specific logic below this line
```

### Hook split convention

| Hook scope | Runs in | Example |
|------------|---------|---------|
| Universal | Every session | Version checks, drift detection, auth validation |
| Git-specific | Git repos only | Worktree listing, issue checks, branch status |

Split monolithic hooks into `*-universal.sh` + `*-git.sh` when they mix both concerns.

## Naming conventions

- Skill directories: `kebab-case` (e.g., `repo-health/`, `git-research/`)
- Skill names in frontmatter: `dc:kebab-case` for global, plain `kebab-case` for project
- Agent files: `kebab-case.md` (e.g., `git-workspace.md`)
- Reference files: `kebab-case.md` describing content (e.g., `label-taxonomy.md`)
- Scripts: `kebab-case.sh` or `snake_case.py`

## WAT framework alignment

This convention extends the WAT framework:

| WAT Layer | Skill component | Role |
|-----------|----------------|------|
| Workflows | SKILL.md + references/ | What good looks like (instructions, patterns) |
| Agents | agent.md + context: fork | Decision-making and orchestration |
| Tools | tools/ | Deterministic execution |

## Sources

- [Claude Code Skills Documentation](https://code.claude.com/docs/en/skills)
- [Skill Authoring Best Practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Claude Code Sub-agents](https://code.claude.com/docs/en/sub-agents)
- [anthropics/skills](https://github.com/anthropics/skills) (official examples)
