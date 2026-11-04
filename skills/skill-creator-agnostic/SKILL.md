---
name: skill-creator-agnostic
description: Creates AI agent skills and rules for any coding agent platform. Defaults to the Agent Skills standard (SKILL.md, read by Claude Code, Copilot, Cursor, Windsurf, OpenCode); generates native rules files (Cursor .mdc, Windsurf rules, Copilot instructions, Aider CONVENTIONS.md) when the user wants always-on rules or targets a tool without skills. Does not activate for general coding questions, debugging, or questions about how an existing skill works.
---

When creating skills: reason as a skill/rule authoring specialist. Generate a standard SKILL.md by default, and platform-native rules files where the target needs them.

**Required before generating:** negative triggers must be defined. Ask if missing.
These rules govern skill file generation only. Follow CLAUDE.md and system prompt for all other output.

## On Invoke

1. Extract skill requirements from user description. Consult with the user to clarify any vague or incomplete requirements. Generate only once all requirements are clear.
2. Detect target platform from context (see Detection), but ask which platform and which output they want: a SKILL.md skill (default) or a rules file.
3. If unclear or multi-platform: ask before generating.
4. Load the relevant platform reference from `references/`.
5. Generate output in that platform's native format.
6. Run discipline review before presenting output (see Before Output).

## Requirements Extraction

Before generating, identify:

- **Objective** — one sentence. If "and" appears, the skill may need splitting — flag it.
- **Triggers** — specific keywords, actions, or file types. Test: could this description match 5 unrelated requests? If yes, it's too vague.
- **Negative triggers** — must be specific (e.g., "does not activate for general coding questions") not generic ("don't activate when not relevant"). Required — define them before generating.
- **Workflow** — step-by-step behavior. Put the most critical step first AND reference it last.
- **Constraints** — hard rules. Soft Cap at 5 hard cap at 8. Reframe any negation: "don't X" → "do Y instead."

Ask if any are unclear. Define negative triggers before generating.

## Platform Detection

Detect from project context before asking:

| Signal | Platform |
|--------|----------|
| `CLAUDE.md` or `.claude/` present, or user mentions Claude Code | Claude Code |
| `opencode.json` or `.opencode/` present, or user mentions OpenCode | OpenCode |
| `.cursor/` present | Cursor |
| `CONVENTIONS.md` or `.aider.conf.yml` present, or user mentions Aider | Aider |
| `.devin/` or `.windsurf/` present (or legacy `.windsurfrules`) | Windsurf |
| `.github/copilot-instructions.md`, `.github/instructions/`, or `.github/skills/` present | Copilot |
| `AGENTS.md` only | Ask which tool reads it |

If ambiguous: ask. If several skill-capable tools are requested: generate one SKILL.md and list the install paths per tool. Generate separate files only for rules-file targets.

## Platform References

Load the target reference before generating. Follow its format exactly.

- Any SKILL.md target → `references/agent-skills.md` (always load first for skills)
- Claude Code extensions → `references/claude-code.md`
- OpenCode → `references/opencode.md`
- Cursor → `references/cursor.md`
- Aider → `references/aider.md`
- Windsurf → `references/windsurf.md`
- Copilot → `references/copilot.md`

Asset templates (blank boilerplate to copy) live in `assets/`.

## Output Rules

- One SKILL.md serves every skill-capable tool. Rules-file targets get one file per platform.
- Use intent-based language ("search the codebase", not tool names like "use Grep").
- Name the output file per platform conventions (see each reference).
- Structure the body: most critical constraint in first 5 lines AND restated at end.
- Rules section in generated output: 5 max, positive framing only.

## Phases

1. **[Requirements]** Gather: platform, objective, triggers, negative triggers, workflow, constraints.
   - On transition: "Requirements confirmed: [platform], [name], [summary]. Generating now."
2. **[Generating]** Load platform reference, generate skill file, run Before Output checklist.
3. **[Review]** Present output. State: "Review the draft. Say 'ship it' to finalize, or describe changes."
4. **[Revising]** Apply only the requested changes. Preserve everything not mentioned. Re-run checklist.
   - State what changed: "Updated: [section]. Unchanged: everything else."
   - Return to Review.
5. **[Done]** On approval or topic change: "Skill complete." Return to default behavior.

## Boundaries

- Activate only when the user requests creating or modifying a skill/rule file for a coding agent platform.
- If the user asks how to use an existing skill: answer directly, do not enter generation mode.
- If the user asks a general coding question unrelated to skill authoring: answer directly.

## Blocking Conditions

- Platform unclear after detection: "I need the target platform. Which one: Claude Code, Cursor, Aider, Windsurf, Copilot, OpenCode? And should I produce a SKILL.md skill or a rules file?"
- Requirements too vague: "These requirements are too broad for a focused skill. I need: [list missing items]."
- User's stated premise seems off: name the discrepancy and ask before proceeding.

## Before Output

Run this review before presenting the generated skill. Fix any failures first.

**Attention**
- [ ] Critical constraint appears in first 5 lines AND restated at end
- [ ] Rules section has 5 or fewer items

**Grounding**
- [ ] All rules use positive framing (no "don't", "never", "avoid")
- [ ] Completion signal present
- [ ] No global behavioral anchors (all anchors scoped to skill's task)

**Robustness**
- [ ] Negative triggers are specific, not generic
- [ ] At least one fallback defined for ambiguous/incomplete input

**Composition**
- [ ] Objective is one responsibility (no "and")
- [ ] Domain declared explicitly in output

**Calibration**
- [ ] No open-ended generation without an uncertainty checkpoint

**Interaction**
- [ ] At least one failure message defined for blocked states

**Security**
- [ ] Skill does not request capabilities beyond its core function

**Agent Skills standard (SKILL.md output)**
- [ ] `name` is lowercase-hyphen, at most 64 chars, and matches the directory
- [ ] `description` is at most 1024 chars and states what it does and when to use it
- [ ] Only the six standard fields unless the user targets a single platform's extensions

Full discipline reference: `../../skill-design-considerations/`

**Reminder: confirm platform, include negative triggers, run checklist before output.**
