# Pattern Library

A living reference of skill design patterns. Each pattern addresses a recurring challenge in skill authoring.

**Format:** Present each pattern as: Name → Problem → Solution → Example → Portability rating.

---

## Pattern: Universal Core / Platform Wrapper

**Problem:** You want a skill that works across multiple agent platforms without rewriting the logic for each one.

**Solution:** Separate your skill into two layers: a universal core (intent, logic, criteria, examples) and a thin platform wrapper (file format, trigger config, tool-specific syntax). Write the core first, then wrap it for each target platform.

**Example:** A code review skill with shared criteria (correctness, security, maintainability) written once as a SKILL.md (read by Claude Code, Copilot, Cursor, Windsurf, and OpenCode), with a condensed CONVENTIONS.md for Aider.

**Portability:** This IS the portability pattern — it's the foundation for all others.

---

## Pattern: Intent Over Mechanism

**Problem:** Skills that name specific tools or commands break when moved to a different agent.

**Solution:** Express what you want accomplished, not how to accomplish it. Say "search the codebase for X" not "use the Grep tool to find X." The agent will map intent to its available tools.

**Example:**
- Mechanism: "Run `git diff --cached` using Bash, then Read each changed file"
- Intent: "Review all staged changes in the current commit"

**Portability:** High — every agent can interpret intent. Only add mechanism when a specific approach is critical for correctness.

---

## Pattern: Progressive Disclosure

**Problem:** Large skill instructions consume context window, reducing the agent's working memory for the actual task.

**Solution:** Layer information by urgency:
- Level 1 (always loaded): Name + description (~100 words)
- Level 2 (on trigger): Core instructions (<500 lines)
- Level 3 (on demand): Detailed references, schemas, examples

**Example:** A database skill with core SQL principles in SKILL.md, detailed schema docs in references/schema.md loaded only when the agent needs them.

**Portability:** Medium — Claude Code supports this natively. On other platforms, approximate by keeping instructions concise and linking to files the agent can read.

---

## Pattern: Activation Boundaries

**Problem:** Skills activate when they shouldn't (wasting tokens) or fail to activate when they should (missing intent).

**Solution:** Define explicit positive AND negative trigger lists. Positive triggers say when to activate. Negative triggers say when to stay quiet — even if keywords match.

**Example:**
```markdown
## Activation Boundaries
* **Active When:**
  * The user asks to create or refactor React components
  * Modifying files matching: `src/components/**/*.tsx`
* **DO NOT ACTIVATE WHEN:**
  * The user is asking a conceptual question without code modification
  * The task targets backend logic (e.g., API routes, database queries)
  * Another specialized skill (testing, deployment) is clearly more appropriate
```

**Portability:** High — every platform benefits from clear activation scope. In always-on rule files (Cursor `alwaysApply`, Windsurf `always_on`, Aider conventions), the agent uses this for self-filtering.

---

## Pattern: Relative Path References

**Problem:** Hardcoded absolute paths break across machines, repos, and platforms. Skills that say `/Users/john/project/src` are useless to anyone else.

**Solution:** Use relative paths or intent-based references. Some platforms have template variables (`{{REPO_ROOT}}` in Cursor), but these are platform-specific — not universal. The most portable approach is relative paths from the project root, or intent-based references that let the agent resolve the path.

**Example:**
- Hardcoded: "Check the file at /home/dev/myapp/src/config.ts"
- Relative: "Check `src/config.ts` for the current configuration"
- Intent-based: "Find and read the project's main configuration file"
- Cursor-specific: "Check `{{REPO_ROOT}}/src/config.ts`" (only works in Cursor)

**Portability:** Relative paths work everywhere. Template variables (`{{REPO_ROOT}}`, `{{CURRENT_FILE}}`) are Cursor-specific — use them in Cursor rules, not in portable skills.

---

## Pattern: Self-Verifying Workflow

**Problem:** The agent completes a task but doesn't check if it actually worked. Errors propagate silently.

**Solution:** Build a verification step into the skill's workflow: after execution, the agent runs a specific check to confirm success before reporting done.

**Example:**
```markdown
## Workflow
1. **Validate:** Check that the target file exists and is a valid TypeScript module
2. **Execute:** Apply the refactoring pattern
3. **Verify:** Run `tsc --noEmit` on the changed file. If it fails, fix the type errors before proceeding.
```

**Portability:** High — the pattern is universal. The specific verification command may differ (npm test, pytest, go vet) but the structure works everywhere.

---

## Pattern: Guardrail Section

**Problem:** Without explicit constraints, agents drift — using deprecated APIs, introducing breaking changes, or violating architectural decisions.

**Solution:** Add a dedicated constraints section with hard rules the agent must never break. Lead with the single most critical guardrail, then list others.

**Example:**
```markdown
## Strict Constraints
> CRITICAL: Never modify files in `src/core/` without explicit user approval.

* Do not use any API deprecated in React 18+
* All new components must be functional (no class components)
* Bundle size increase > 5KB requires justification
```

**Portability:** High — constraints are pure logic. They work identically on every platform.

---

## Pattern: Minimal Tool Assumptions

**Problem:** Different agents expose different tools with different names and capabilities.

**Solution:** Assume only the universal capabilities: read files, write/edit files, search content, execute commands. Don't assume named tools exist. When a specific tool IS required, document it as a dependency.

**Example:**
```markdown
## Dependencies
This skill requires:
- File search capability (any form)
- File read/write capability
- Command execution (for running tests)

Optional (enhances but not required):
- Web search (for documentation lookup)
- Memory/state persistence (for progress tracking)
```

**Portability:** High — explicitly declaring dependencies makes porting straightforward.
