# Cursor Format

**Prefer a SKILL.md skill** (`.cursor/skills/<name>/` or `.agents/skills/<name>/`; format in `references/agent-skills.md`). Cursor applies skills when the agent finds them relevant, and `disable-model-invocation: true` makes one behave like a slash command. Use a rules file (below) for always-on or file-scoped guidance.

## Rules File Location

`.cursor/rules/skill-name.mdc`. Plain `.md` files in that folder are ignored. `AGENTS.md` at the repo root (or in subdirectories) is a frontmatter-free alternative.

MDC supports per-file scoping via `globs`.

## Frontmatter (MDC only)

```yaml
---
description: One sentence. Used by Cursor's agent mode to select rules.
globs: ["src/**/*.ts", "tests/**/*"]   # Files that auto-attach this rule
alwaysApply: false                      # true = always in context, false = on-demand
---
```

**`globs`** — Cursor auto-attaches rules matching open files. Omit if skill should only activate manually.
**`alwaysApply: true`** — loads into every chat. Use sparingly — burns context on every request.

## Rule Types

| Type | Trigger |
|------|---------|
| Always Apply | Every chat session (`alwaysApply: true`) |
| Apply Intelligently | Agent reads `description` and decides |
| Apply to Specific Files | `globs` match |
| Apply Manually | `@`-mention in chat |

Context variables such as `{{REPO_ROOT}}` do not appear in current Cursor docs; use relative paths instead.

## Body Format

Structured sections work well in Cursor — it renders markdown in the rule panel.
Emoji headers are optional but common in the Cursor ecosystem.

Recommended structure:

```markdown
## Activation Boundaries
**Active when:**
- User asks to [action]
- Editing files matching `[pattern]`

**Do NOT activate when:**
- User asks a theoretical/conceptual question
- Task targets a different stack (e.g., backend logic in a frontend rule)

## Context & Objective
[Persona + high-level goal. E.g., "You are an expert in Next.js 15 App Router..."]

## Workflow
1. **Validate** — check before writing code
2. **Execute** — concrete steps and preferred commands
3. **Verify** — how to test the output

## Constraints
- [Hard rule 1]
- [Hard rule 2]

## Reference Examples
[Minimal idiomatic code snippet to anchor correct output]
```

## Negative Triggers

Include a "Do NOT activate when" section. Keep each rule under 500 lines and split large rulesets. This is the primary token-saving mechanism in Cursor — rules that fire when they shouldn't bloat every prompt.

## Template

See `assets/cursor-template.mdc`
