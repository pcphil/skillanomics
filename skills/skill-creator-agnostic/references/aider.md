# Aider Convention Format

## File Location

**Primary:** `CONVENTIONS.md` in project root
**Config-referenced:** Any file listed under `read:` in `.aider.conf.yml`

```yaml
# .aider.conf.yml
read:
  - CONVENTIONS.md
  - docs/architecture.md
```

Files in `read:` (or `aider --read CONVENTIONS.md`) are loaded read-only and cached when prompt caching is on. Keep them focused — Aider's docs show no lazy loading, and no skills (SKILL.md) support was found. `agents.md` lists Aider among tools that read `AGENTS.md`; confirm for your version.

## Format

Plain markdown. No frontmatter. No special variables. No emoji sections.

Aider reads conventions as background context, not as executable instructions. Write for a senior developer scanning quickly — not for an agent parsing structured steps.

## Recommended Structure

```markdown
# Project Conventions

## Stack
[Language, framework, versions — pin exact versions to prevent hallucination]

## Code Style
[Naming conventions, file structure, import order]

## Patterns to Follow
[Preferred abstractions, architecture patterns, idioms]

## Patterns to Avoid
[Anti-patterns, forbidden approaches, deprecated APIs]

## Commit Style
[Commit message format, branch naming, PR conventions]

## Testing
[Test framework, file location, naming, coverage expectations]
```

## Negative Triggers

Aider has no activation boundary system. Negative guidance goes in "Patterns to Avoid":

```markdown
## Patterns to Avoid
- Do not use [pattern] — use [alternative] instead.
- Do not modify files in `generated/` — they are auto-generated.
```

## Git Awareness

Aider is deeply git-integrated. Include:
- Commit message format (conventional commits, etc.)
- Whether to amend or create new commits
- Branch naming conventions
- Files that should never be committed (`.env`, secrets)

## Size

Aider documents no size limit, but the full file loads every session. Keep it lean (about 100 lines). Move detail to separate files and reference them via `.aider.conf.yml`.

## Template

See `assets/aider-template.md`
