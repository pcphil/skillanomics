# Positive Constraint Grammar

## What it is

Every constraint in a skill should be expressed as what to do, not what not to do. "Confirm platform before generating" names the action. "Don't generate without knowing the platform" only names what to avoid. The positive form gives the model a concrete action to take; the negative form leaves the model to infer the right action, and is followed less dependably under ambiguity or in long contexts.

This principle operationalizes negation-failure (grounding/01) as a writing rule: apply it during authoring, not just as a retrospective fix.

## Why it matters

Negative constraints have two failure modes:
1. **Suppression failure** — the model generates the prohibited thing anyway, especially in ambiguous conditions or long contexts
2. **Gap failure** — "don't do X" leaves the correct behavior undefined; the model fills the gap with its default, which may be exactly X

Positive constraints sharply reduce both failure modes: the model has a concrete target behavior, and the gap between "what to do" and "what not to do" disappears.

Beyond reliability: positive constraints are also easier to read and audit. A constraint audit should be able to confirm "the model knows what to do in this case" — not just "the model knows what to avoid."

## How to apply

**Audit every constraint before shipping:**

Read through the Rules, Constraints, and Boundaries sections of the skill. For each item:
- Does it start with "don't", "never", "avoid", "no", "without"?
- If yes: rewrite it as a positive action.

**Rewrite table:**

| Negative (weaker) | Positive (stronger) |
|-----------------------|---------------------|
| Don't generate without knowing the platform | Confirm platform before generating |
| Never use tool names in platform-portable skills | Use intent-based language ("search the codebase") |
| Avoid activating for general questions | Activate only when a skill file is provided or creation is requested |
| Don't blend output from multiple platforms | Generate one platform's output at a time, labeled clearly |
| Never fabricate version numbers | State version numbers only when certain; use [verify in docs] otherwise |
| Don't ask more than 2 questions | Ask at most 2 questions; then generate with stated assumptions |
| Avoid delivering output before confirmation | Confirm with user before finalizing output |

**Handle genuine out-of-scope boundaries separately:**

Some constraints genuinely need to express "this is not something this skill does." These belong in a `## Boundaries` or `## Out of Scope` section — and each negative there should be paired with a redirect:

```markdown
## Boundaries
- This skill does not review general code (only SKILL.md files).
  For code review: use the code-reviewer skill or ask directly.
- This skill does not generate platform-specific IDE configurations.
  For those: use the platform's native config tooling.
```

The redirect converts the negative boundary into a positive action for the user.

**Apply positive framing at the decision point:**

A positive constraint placed immediately before the action it governs is more reliable than one stated only at the top of the file:

```markdown
# Stated only at top (weaker)
## Rules
- Confirm platform before generating

## Workflow
1. Gather requirements
2. Generate output  ← no reminder here; model may skip confirmation

# Embedded at decision point (stronger)
## Workflow
1. Gather requirements
2. Confirm platform — required before proceeding to generation
3. Generate output
```

## Example

**Before — negative constraints throughout:**

```markdown
## Rules
- Don't generate without knowing the target platform
- Never use Claude Code tool names in a Cursor skill
- Don't activate for general questions about skill design
- Never produce output longer than 150 lines
- Don't ask more than 2 clarifying questions
- Avoid making assumptions without stating them
- Never ignore the user's negative trigger examples
```

Each rule tells the model what to avoid. None tells it what to do instead. All are vulnerable to suppression failure.

**After — positive constraint grammar:**

```markdown
## Rules
- Confirm target platform before generating any output
- Use intent-based language in platform-portable skills ("search the codebase")
- Activate when: a skill file is provided for review, or skill creation is explicitly requested
- Keep skill body under 150 lines; move detail to references/
- Ask at most 2 clarifying questions; generate with stated assumptions after that
- State assumptions explicitly before acting on them
- Incorporate the user's negative trigger examples into the skill description

## Boundaries
Outside this skill's scope:
- General skill design questions (not about a specific file) → suggest skill-tutor
- Non-skill-file review requests → ask if user meant to provide a different input
```

Same constraints. All positive. Each gives the model a concrete action. The boundaries section uses redirect pairs.
