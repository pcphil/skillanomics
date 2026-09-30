# Instruction Hierarchy Blindness

## What it is

Agent environments layer multiple sources of instructions: system prompt, user-level config (CLAUDE.md), loaded skills, and the active user message. These sources have a rough precedence — system prompt is most authoritative, followed by user config, then skills. User messages sit below the system prompt in the designed hierarchy, but their recency gives them high practical weight, so in practice they often override a skill. A skill that ignores this layering will be silently overridden without any indication of conflict, causing it to behave inconsistently across environments.

This is distinct from instruction conflict (grounding/03), which covers clashes between two specific instructions. Hierarchy blindness is a design-time omission: the skill was written as if it's the only instruction source, with no consideration of where it sits in the stack.

## Why it happens

Skill authors test in a controlled environment: one skill, a minimal system prompt, a clean CLAUDE.md. In that context, the skill is the dominant instruction source and behaves as written. When the same skill is deployed into a richer environment — a customized system prompt, a CLAUDE.md full of project conventions, user messages with strong imperatives — the skill's instructions get displaced and the author never sees why.

The model resolves conflicts silently. It doesn't announce "your skill said X but the system prompt said Y so I'm doing Y." It just does Y.

## Analogy

A new employee writes a detailed process guide for how to handle customer calls. It's excellent. But the company's employee handbook (written by HR, mandatory) already specifies call handling — and it says something different. On their first day, their manager (a user message) tells them something different again. The new employee's guide is now third in line. It won't be followed in any conflict. Nobody told them this when they wrote it.

## Symptoms

- Skill behaves correctly when tested in isolation but breaks in the user's actual environment
- Formatting, tone, or workflow instructions from the skill are ignored — user's CLAUDE.md has a conflicting rule
- Skill's constraints are overridden when the user phrases a message as a strong imperative ("just do it, no questions")
- Skill works in one user's environment but not another's (different system prompts)
- "IMPORTANT" or "REQUIRED" in caps gets added to win conflicts — this doesn't change the skill's position in the stack, and on recent Claude models (per Anthropic's prompting guidance) it tends to cause overtriggering (the rule fires in cases it wasn't meant for)

## Fix

**Know your position in the stack:**

Assume this rough precedence (most authoritative to least), noting the last item is recency-weighted rather than strictly ranked:
1. System prompt (set by platform/operator — highest authority)
2. User config (CLAUDE.md, settings)
3. Active skill (you)
4. User message (below the system prompt by design, but recency often makes it win in practice — don't assume the skill outranks it)

This is a practical ordering, not how the content is delivered. In Claude Code, for example, CLAUDE.md is injected as user-turn context rather than system prompt, and a skill body enters mid-conversation only when the skill activates. An explicit, direct user instruction usually wins over a skill. Treat the list as "who tends to win a conflict," and verify on each target platform.

Design the skill to function within this hierarchy, not above it.

**Declare override sensitivity:**

State explicitly which of the skill's rules are negotiable vs. non-negotiable, and how the model should handle conflicts:

```markdown
## Conflict Handling
These rules govern this skill's output only. If CLAUDE.md or the system prompt
specifies a different convention, follow those for non-skill output.
Core workflow (ask → generate → confirm) is not overridable by user messages alone.
```

**Test across environments:**

Before publishing, test the skill with:
- A non-empty CLAUDE.md (common project conventions active)
- A user message that directly contradicts a skill rule
- A system prompt that specifies output format preferences

**Anchor the non-negotiable minimum:**

If one rule must hold regardless of environment, state it in the first 5 lines of the skill body — and state why it's required, not just what it is. Rationale survives hierarchy conflicts better than bare commands.

## Example

**Bad — assumes full authority:**

```markdown
## Rules
- Always ask for the target platform before generating
- Format all output with YAML frontmatter
- Never produce more than 300 lines
- Confirm with user before finalizing
```

These read as global policy. They'll lose silently to any CLAUDE.md or system prompt that contradicts them.

**Good — hierarchy-aware:**

```markdown
## Rules
These rules govern skill file generation only. They may coexist with broader
conventions from CLAUDE.md or the system prompt — follow those for other output.

- Before generating a skill file: confirm target platform (required for correctness)
- Skill files use YAML frontmatter — other output follows whatever format is active
- Skill file body: aim for under 300 lines; move excess to references/
- After generating: ask user to confirm before marking task complete

## Conflict Handling
If a user message contradicts the platform-confirmation step, explain why it
matters and ask once more before proceeding.
```

Scoped rules survive hierarchy conflicts. Rationale-backed rules survive user pressure.
