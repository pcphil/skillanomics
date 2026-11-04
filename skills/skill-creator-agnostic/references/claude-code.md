# Claude Code Skill Format

## File Location
```
skill-name/
├── SKILL.md          # Required. Entry point.
├── references/       # On-demand detail — load when needed
├── scripts/          # Deterministic executables
└── assets/           # Templates/boilerplate — not loaded into context
```

Install: place `skill-name/` under `.claude/skills/` (project) or `~/.claude/skills/` (personal). `.agents/skills/` is also widely read. Layout and standard fields: `references/agent-skills.md`.

## Frontmatter

```yaml
---
name: kebab-case-name
description: >
  One or two sentences. Dense with trigger keywords — this drives skill matching.
  Include the slash command if one exists (e.g., "invokes /skill-name").
---
```

**`name` and `description` are the portable core.** Claude Code accepts extra fields (`when_to_use`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `context`, `paths`, `hooks`, `model`, `effort`, and more), but other tools and claude.ai uploads accept only the six standard fields (`name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`). Add extensions only when the skill needs them and the user targets Claude Code.

## Body Format

Plain prose markdown. No emojis. No emoji section headers.

Structure:
1. **On Invoke** — what to do first (check memory, detect state, ask questions)
2. **Core workflow** — step-by-step behavior
3. **Rules** — constraints, written as positive instructions where possible
4. **Boundaries** — explicit out-of-scope statements

## Tool References

Name tools explicitly — Claude Code executes these directly:

- `Read` — read a file
- `Grep` — search codebase
- `Glob` — find files by pattern
- `Edit` / `Write` — modify/create files
- `Bash` / `PowerShell` — run commands
- `AskUserQuestion` — ask user structured questions with options
- Memory system — persist state across sessions

## Negative Triggers

Claude Code doesn't have a dedicated negative trigger field. Embed in description or body:

```markdown
## Boundaries
- Only activate for X. For Y: say "out of scope, focus on X."
- Skip if user is asking a conceptual question without code modification.
```

## State & Memory

Track progress across sessions using Claude Code memory (type: `project`):

```markdown
Save to memory: current step, what's been built, user background.
On resume: read memory → summarize where they left off → ask continue or restart.
```

## Size Constraint

Keep SKILL.md under 500 lines. Combined `description` + `when_to_use` is capped at 1,536 characters. Move heavy content to `references/`. Reference it explicitly:

```markdown
Full spec: `references/detail.md`
```

## Template

See `assets/claude-code-template.md`
