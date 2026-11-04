# Module 1: Portability Patterns

## Lesson 1.1 — What "Portable" Means

### Concept

A portable skill is one whose core logic and intent can be expressed on any coding agent platform with minimal rewriting. Portability doesn't mean "write once, run everywhere" — it means **designing so the expensive part (the thinking) is reusable**, even when the packaging changes.

Every coding agent, regardless of platform, does three things:
1. Reads instructions (system prompt, rules file, skill body)
2. Has access to tools (file read/write, search, execute, web)
3. Operates within a context window (limited attention)

These three universals are your portability surface. A skill that's built around *what to think about* rather than *how to invoke a specific tool* will translate across platforms.

**Why this matters even if you only use Claude Code today:**
- You might switch tools. Your skill investment shouldn't be lost.
- Skills designed for portability tend to be cleaner — they force you to separate intent from mechanism.
- The agent landscape is moving fast. Today's platform may merge features from others.

### Example

**Non-portable instruction:**
```
When the user invokes /review, use the Grep tool to find all TODO comments,
then use the Read tool to load each file, and output findings using markdown.
```

**Portable instruction:**
```
When reviewing code, find all TODO comments across the project, read the
surrounding context for each, and present a summary grouped by priority.
```

The second version expresses *intent*. Any agent can figure out how to search and read files. The first version is Claude Code-specific (tool names, invocation pattern).

### Exercise

Look at this instruction and rewrite it to be portable:
> "Use the Bash tool to run `npm test`, capture the output, then use Edit to fix any failing assertions."

What's the intent? What's platform-specific?

---

## Lesson 1.2 — The Agent Interface Landscape

### Concept

Two kinds of files exist across coding agents, and they behave differently:

- **Skills** (`SKILL.md` in a folder): the open **Agent Skills** standard (agentskills.io). Only `name` and `description` load at startup; the body loads when the skill activates; `references/`, `scripts/`, and `assets/` load on demand. Claude Code, GitHub Copilot, Cursor, Windsurf (Devin Desktop), OpenCode, and many other tools read this format.
- **Rules / instruction files**: always-on or pattern-scoped markdown. Each tool has its own file (`.cursor/rules/*.mdc`, `.windsurf/rules/*.md`, `.github/copilot-instructions.md`, `CONVENTIONS.md`), plus the cross-tool **`AGENTS.md`** (plain markdown at the repo root, nearest file wins).

*Verified against vendor docs, September 2026. Tool docs change quickly: recheck before relying on a path.*

| Platform | Skills (SKILL.md) | Rules / instructions | AGENTS.md |
|----------|-------------------|----------------------|-----------|
| Claude Code | `.claude/skills/`, `~/.claude/skills/`, plugins | `CLAUDE.md` | Not the primary file (uses `CLAUDE.md`) |
| GitHub Copilot | `.github/skills/`, `.claude/skills/`, `.agents/skills/` | `.github/copilot-instructions.md`; `.github/instructions/*.instructions.md` with `applyTo` globs | Yes (also reads `CLAUDE.md`, `GEMINI.md`) |
| Cursor | `.cursor/skills/`, `.agents/skills/` (legacy `.claude/skills/`) | `.cursor/rules/*.mdc` (`description`, `globs`, `alwaysApply`) | Yes |
| Windsurf (Devin Desktop) | `.devin/skills/`, `~/.codeium/windsurf/skills/` | `.devin/rules/*.md` or `.windsurf/rules/*.md` (`trigger:` modes) | Yes (directory-scoped) |
| OpenCode | `.opencode/skills/`, `.claude/skills/`, `.agents/skills/` | `AGENTS.md` | Yes |
| Aider | No skills support found in its docs | `CONVENTIONS.md` loaded via `--read` or `.aider.conf.yml` (`read:`) | Listed as supported at agents.md; confirm for your version |

**Key differences:**
- **Skills are the portable unit.** A `SKILL.md` with `name` + `description` works across the skill-capable tools above. Extra Claude Code fields (`disable-model-invocation`, `context`, `paths`, `hooks`, and others) are extensions; tools that follow the standard may reject or ignore them. claude.ai uploads accept only the standard fields.
- **Rules are per tool.** Trigger modes differ: Cursor uses `alwaysApply`, description, or `globs`; Windsurf uses `always_on`, `model_decision`, `glob`, or `manual`; Copilot uses `applyTo`.
- **Size guidance differs:** Claude Code and Cursor say keep files under 500 lines; Windsurf caps rule files at 12,000 characters; Copilot says instructions should be no longer than 2 pages. The Agent Skills spec recommends SKILL.md instructions under about 5,000 tokens and 500 lines.
- **Tool naming:** every platform names its tools differently. Use intent-based language.
- **Memory:** Claude Code memory is not a universal feature. Design a fallback.

**Key similarities:**
- All read markdown instructions
- All can search, read, and modify files
- All benefit from clear, concise instructions with examples

### Exercise

Pick two platforms from the table. List three things they share and three things that would need adaptation when porting between them.

---

## Lesson 1.3 — Universal Core, Thin Wrapper

### Concept

Write the **core** once as a standard `SKILL.md`. Adapt only where a platform lacks skills or a feature.

```
┌─────────────────────────────┐
│     Platform Adapter         │  ← Only for tools without skills (Aider) or extra features
├─────────────────────────────┤
│  SKILL.md (Agent Skills)     │  ← Intent, workflow, references/, scripts/
└─────────────────────────────┘
```

**In the SKILL.md (portable across skill-capable tools):**
- `name` (matches directory, lowercase, hyphens, max 64 chars) and `description` (max 1024 chars, keyword-rich, with negative triggers)
- Workflow, rules, boundaries in intent-based language
- `references/`, `scripts/`, `assets/` with relative paths

**In an adapter (only when needed):**
- An always-on rules file (`.cursor/rules/*.mdc`, `.windsurf/rules/*.md`, `CONVENTIONS.md`) that carries a short version of the core, or points to it
- Platform features (Claude Code `allowed-tools`, hooks, memory) with a documented fallback

### Example: Code Review Skill

**Core** (`skills/code-review/SKILL.md`, works in every skill-capable tool):
```yaml
---
name: code-review
description: Reviews code for correctness, security, and maintainability. Use when the user asks for a review or feedback on changes. Does NOT activate for writing new code.
---
```
Body: review criteria, output format (summary, issues, strengths), references for detailed checklists.

**Adapter for Aider** (no skills): a `CONVENTIONS.md` with the criteria condensed to a short list, loaded with `--read`.

Same brain, near-identical packaging where the standard is supported.

---

## Lesson 1.4 — Mapping Between Platforms

### Concept

| Concept | SKILL.md tools (Claude Code, Copilot, Cursor, Windsurf, OpenCode) | Rules-only files (Aider; Cursor/Windsurf/Copilot rule files) |
|---------|-------------------------------------------------------------------|---------------------------------------------------------------|
| Identity | `name` frontmatter | File name |
| Trigger text | `description` frontmatter | Rule-specific: `description` + `alwaysApply` (Cursor), `trigger:` (Windsurf), `applyTo` (Copilot), none (Aider) |
| File-scoped activation | `paths` (Claude Code, Cursor extensions) | `globs` (Cursor), `glob` trigger (Windsurf), `applyTo` (Copilot) |
| Extended knowledge | `references/` loaded on demand | Inline, or extra files loaded with `--read` (Aider) |
| Deterministic actions | `scripts/` | Terminal commands |
| Context management | Progressive disclosure (metadata, body, resources) | Keep the file short: it loads every time it applies |
| State persistence | Memory (Claude Code only); fallback below | File-based state or statelessness |

**Handling missing capabilities:**
- **No skills support** (Aider) → Put a condensed core in the conventions file; keep it short.
- **Always-on rule files** → Add a "When to activate" section so the agent self-filters.
- **No memory** (everything except Claude Code) → Write progress to a small project file (for example `.inspiration-state.md`, git-ignored) or accept statelessness.
- **Extension fields rejected** (claude.ai uploads, strict Agent Skills validators) → Keep to the six standard fields (`name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`) for distributable skills.
- **Limited tools** → Express intent and let the agent use what it has.

---

## Lesson 1.5 — Anti-Patterns

### Hard-Coded Tool Names
```
BAD:  "Use the Grep tool to search for..."
GOOD: "Search the codebase for..."
```
The agent knows how to search. Tell it *what* to find, not *which tool to use* — unless a specific tool is critical for correctness.

### Platform Path Assumptions
```
BAD:  "Read ~/.claude/memory/progress.md to check status"
GOOD: "Check learning progress from your persistent state"
```

### Over-Specified Invocation
```
BAD:  "When the user types /review followed by a file path..."
GOOD: "When the user requests a code review..."
```
Slash commands are a Claude Code convention. The universal intent is "user wants a review."

### Portability Theater
Don't abstract for portability if:
- You genuinely only use one platform and don't plan to change
- The skill relies heavily on platform-specific features (hooks, MCP servers)
- The overhead of abstraction exceeds the skill's complexity

Portability is a design principle, not a religion. Apply it where it earns its keep.
