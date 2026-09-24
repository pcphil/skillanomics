# GitHub Copilot Format

**Prefer a SKILL.md skill** for task-specific workflows (`.github/skills/`, `.claude/skills/`, or `.agents/skills/`; format in `references/agent-skills.md`). Use instruction files for always-on conventions.

## Instruction File Locations

| File | Scope |
|------|-------|
| `.github/copilot-instructions.md` | Repository-wide, always applied |
| `.github/instructions/NAME.instructions.md` | Path-specific, via `applyTo` globs in frontmatter |
| `AGENTS.md` (also `CLAUDE.md`, `GEMINI.md` at the root) | Agent instructions; nearest file wins |

Path-specific frontmatter:

```yaml
---
applyTo: "**/*.ts,**/*.tsx"
---
```

## Format

Repository-wide file: plain markdown, no frontmatter. Write in direct, imperative prose.

## What Instructions Do Well

- Set coding style, naming, and conventions
- Specify preferred libraries, patterns, and version constraints
- Document build, test, and validation commands and project layout
- Scope rules to paths with `applyTo`

## Limits

- No dynamic file references or variables
- No persistent state across sessions
- Repository-wide instructions always apply; use `applyTo` files or a skill for narrower activation

## Recommended Structure

```markdown
# Copilot Instructions

## Stack
- Language: [TypeScript 5.x / Python 3.12 / etc.]
- Framework: [Next.js 15 / FastAPI / etc.]
- Test framework: [Vitest / pytest / etc.]

## Code Style
- [Naming convention]
- [Import style]

## Preferred Patterns
- [Pattern 1]

## Patterns to Avoid
- [Anti-pattern 1]

## Testing
- [Test location, naming, run command]
```

## Negative Triggers

Bound scope with `applyTo` globs, or with a Scope section:

```markdown
## Scope
These instructions apply to `src/` only, leaving `scripts/` and `infra/` untouched.
```

## Size

GitHub's guidance: instructions must be no longer than 2 pages. Put the highest-impact conventions first.

## Template

See `assets/copilot-template.md`
