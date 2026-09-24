# OpenCode Skill Format

## File Placement

Skills load from these locations (all valid):

```
.opencode/skills/<name>/SKILL.md           # project-level
~/.config/opencode/skills/<name>/SKILL.md  # global
.claude/skills/<name>/SKILL.md             # Claude-compatible path
.agents/skills/<name>/SKILL.md             # agent-compatible path
```

Directory layout within each skill:

```
skill-name/
├── SKILL.md          # Required. Entry point.
├── references/       # On-demand detail
├── scripts/          # Deterministic executables
└── assets/           # Templates/boilerplate — not loaded into context
```

## Discovery

OpenCode traverses upward from the current working directory to the git worktree root, loading matching skill files. Global definitions are also sourced from home directory locations.

## Frontmatter

```yaml
---
name: kebab-case-name           # required
description: >                   # required, 1–1024 chars
  One or two sentences rich in trigger keywords.
license: MIT                     # optional
compatibility: Designed for OpenCode   # optional, string
metadata:                        # optional, key-value pairs
  key: value
---
```

**Required:** `name` and `description` only. Rest are optional.

## Name Rules

- Pattern: `^[a-z0-9]+(-[a-z0-9]+)*$`
- Length: 1–64 characters
- Lowercase alphanumeric, single hyphens as separators
- No leading/trailing hyphens, no consecutive hyphens
- Must match parent directory name exactly

## Permission Control

Configure via `opencode.json` using glob patterns:

```json
{
  "permission": {
    "skill": {
      "internal-*": "allow",
      "experimental-*": "deny",
      "dangerous-*": "ask"
    }
  }
}
```

- `allow` — load immediately without prompting
- `deny` — hidden from agents entirely
- `ask` — user approval required before loading
- Patterns support wildcards (e.g., `internal-*`)

## Agent-Specific Overrides

Custom agents define permissions in frontmatter. Built-in agents use configuration sections under their agent names in `opencode.json`.

## Disabling Skills

Set `tools: { skill: false }` in agent frontmatter or configuration to disable skill tool access for that agent.

## Body Format

Plain prose markdown. Same structure as Claude Code:

1. **On Invoke** — what to do first
2. **Core workflow** — step-by-step behavior
3. **Rules** — positive constraints
4. **Boundaries** — explicit out-of-scope statements

## Key Differences from Claude Code

| Topic | Claude Code | OpenCode |
|-------|-------------|----------|
| Extra frontmatter | Many extensions (`disable-model-invocation`, `context`, `paths`, and more) | Standard fields: `license`, `compatibility`, `metadata` |
| Permission system | None (all skills available) | `allow`/`deny`/`ask` via `opencode.json` |
| Tool names | `Read`, `Grep`, `Glob`, `Edit`, `Write` | Different tool set — use intent-based language |
| Memory system | Persistent across sessions | Verify support before relying on it |
| Disable skills | N/A | `tools: { skill: false }` in agent config |

**Use intent-based language** in skill bodies — say "search the codebase" not "use the Grep tool". Keeps skills portable.

## Portability Note

A well-written OpenCode skill and a well-written Claude Code skill should look nearly identical. If they don't, the skill is probably too platform-specific.

## Troubleshooting

Skill not appearing? Check:

1. Filename capitalization — must be `SKILL.md` (uppercase)
2. Required frontmatter fields present (`name`, `description`)
3. Name uniqueness across all locations (no duplicates)
4. `name` field matches parent directory name exactly
5. Permission settings in `opencode.json` not blocking it

## Template

See `assets/opencode-template.md`
