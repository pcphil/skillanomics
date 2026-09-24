# Agent Skills Standard (SKILL.md)

The default target. One `SKILL.md` folder is read by Claude Code, GitHub Copilot, Cursor, Windsurf (Devin Desktop), OpenCode, and many other tools. Spec: agentskills.io. Verified against vendor docs, September 2026; recheck paths before relying on them.

## Layout

```
skill-name/
├── SKILL.md          # Required. Frontmatter + instructions.
├── references/       # On-demand detail
├── scripts/          # Deterministic executables
└── assets/           # Templates, not loaded into context
```

## Frontmatter

```yaml
---
name: kebab-case-name        # required; must match the directory name
description: >               # required; 1-1024 chars; what it does AND when to use it
  Keyword-rich sentences. State negative triggers: "Does NOT activate for ...".
license: MIT                 # optional
compatibility: Needs git     # optional; string, max 500 chars
metadata:                    # optional; string keys and values
  version: "1.0"
allowed-tools: Bash(git:*)   # optional; experimental; space-separated
---
```

**`name` rules:** lowercase letters, digits, hyphens; max 64 chars; no leading, trailing, or double hyphen.

**Distributable skills:** keep to these six fields. Platform extensions (Claude Code's `disable-model-invocation`, `context`, `paths`, `hooks`, and others) work only where supported; claude.ai uploads reject them.

## Progressive Disclosure

1. Metadata (`name`, `description`) loads at startup for every skill.
2. The SKILL.md body loads when the skill activates. Keep it under 500 lines (about 5,000 tokens).
3. `references/`, `scripts/`, `assets/` load on demand. Keep references one level deep.

## Where Each Tool Looks

| Tool | Project paths |
|------|---------------|
| Claude Code | `.claude/skills/<name>/` |
| GitHub Copilot | `.github/skills/`, `.claude/skills/`, `.agents/skills/` |
| Cursor | `.cursor/skills/`, `.agents/skills/` (legacy `.claude/skills/`) |
| Windsurf (Devin Desktop) | `.devin/skills/` |
| OpenCode | `.opencode/skills/`, `.claude/skills/`, `.agents/skills/` |

`.agents/skills/` and `.claude/skills/` are the most widely read shared locations.

## Negative Triggers

No dedicated field. Put them in `description` ("Does NOT activate for ...") and in a Boundaries section.

## Template

See `assets/claude-code-template.md` (works for any SKILL.md target; delete the memory lines if the target lacks memory).
