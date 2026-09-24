# Windsurf (Devin Desktop) Format

**Prefer a SKILL.md skill** for multi-step work with supporting files (`.devin/skills/<name>/`; format in `references/agent-skills.md`). Invoke automatically by description or with `@skill-name`. Use a rules file for short behavioral guidance.

## Rules File Location

- Workspace rules: `.devin/rules/*.md` (preferred) or `.windsurf/rules/*.md`
- Global rules: `~/.codeium/windsurf/memories/global_rules.md`
- `AGENTS.md` in any workspace directory (no frontmatter, always on)
- Legacy: `.windsurfrules` in the project root (older format; not covered by current docs)

## Format

Markdown with optional frontmatter in workspace rules:

```yaml
---
trigger: model_decision        # always_on | model_decision | glob | manual
description: Shown to the model when deciding relevance (model_decision)
globs: "src/**/*.ts"           # for trigger: glob
---
```

| Trigger | Behavior |
|---------|----------|
| `always_on` | Full content in the system prompt every message |
| `model_decision` | Description shown; full content loaded when relevant |
| `glob` | Applied when matching files are accessed |
| `manual` | Activated by `@rule-name` |

Global rules and root `AGENTS.md` have no frontmatter and are always on. Write instructions in second person, present tense.

## Recommended Structure

```markdown
# [Project Name] Rules

## Role & Objective
You are [persona]. Your primary goal is [objective].

## Stack
- Language: [e.g., TypeScript 5.x]
- Framework: [e.g., Next.js 15 App Router]

## Workflow
1. [Step 1]
2. [Step 2]

## Always
- [Positive constraint]

## Out of Scope
[What this rule does NOT cover — prevents drift]
```

## Negative Triggers

Use `trigger: glob` or `model_decision` to bound when a rule loads, and an "Out of Scope" section to bound behavior:

```markdown
## Out of Scope
- Backend code: this project is frontend only.
- Files in `src/generated/`: auto-generated.
```

## Size

Workspace rule files are limited to 12,000 characters each, and global rules to 6,000. Shorter files follow instructions more reliably.

## Template

See `assets/windsurf-template.md`
