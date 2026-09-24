# Skills

A **skill** is a markdown prompt loaded into a coding agent to give it specialised behaviour — a tutor, a code reviewer, a scaffolder, etc.

## Structure

```
skill-name/
├── SKILL.md          # Required. Frontmatter + instructions.
├── references/       # Heavy docs loaded on demand.
├── scripts/          # Deterministic helper scripts.
└── assets/           # Templates and boilerplate.
```

**SKILL.md** is the entry point. It uses YAML frontmatter for metadata and markdown for instructions:

```markdown
---
name: skill-name
description: One or two sentences — drives trigger matching and search.
---
# Instructions...
```

## Activation

Copy or symlink a skill directory into `~/.agents/skills/` (or `.claude/skills/` for project-scoped skills) and reload plugins. The `description` field controls when the agent auto-activates the skill.

## Available Skills

| Skill | Description |
|-------|-------------|
| [learn-python](learn-python/) | Guided Python learning — zero to hero through real-world projects. Foundations, intermediate, and domain tracks (Web, Data, CLI, AI). |
| [learn-react](learn-react/) | Guided React learning — components, state, hooks, and building real UIs. |
| [learn-typescript](learn-typescript/) | Guided TypeScript learning — type system fundamentals through generics to advanced types. Teaches via Concept → Analogy → Workshop loop, with each workshop verified by `tsc --noEmit --strict`. |
| [learn-dsa](learn-dsa/) | Guided Data Structures & Algorithms in Python — Big-O through graphs. Teaches via Concept → Analogy → Workshop loop with LeetCode-style problems. |
| [skill-tutor](skill-tutor/) | Teaches how to build optimal, portable agent skills. Tutor, Reviewer, and Librarian modes. |
| [skill-creator-agnostic](skill-creator-agnostic/) | Scaffolds new skills as standard SKILL.md (Claude Code, Copilot, Cursor, Windsurf, OpenCode) or native rules files (Cursor, Windsurf, Copilot, Aider). |
| [inspiration](inspiration/) | Brainstorming partner that grills you like a product manager and scans an existing codebase (app, web, or API) for evidence-backed quality-of-life ideas and new feature opportunities, ending with a user-approved spec. |

## Best Practices

- **Keep SKILL.md short** — under 500 lines. Move heavy content to `references/` and load it on demand.
- **Write the description carefully** — it is used for semantic search and trigger matching; keyword-rich and specific beats vague.
- **One concept per skill** — a skill that does one thing well is easier to trigger correctly and easier to maintain.
- **Express intent, not mechanism** — write "search the codebase for X" not "use the Grep tool". This keeps skills portable across agents.
- **Define clear boundaries** — state explicitly when the skill should *not* activate to avoid wasting context on irrelevant tasks.
- **Prefer flat structure** — avoid deep nesting inside a skill directory; it adds complexity without benefit.
