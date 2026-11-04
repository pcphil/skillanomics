# Negation Failure

## What it is

Instructions phrased as prohibitions ("don't do X", "never do Y", "avoid Z") are less dependable than instructions phrased as positive actions ("do A instead", "always do B", "use C"). Current models follow simple, clearly stated prohibitions well; the gap shows up when a skill relies heavily on negatives, when conditions are ambiguous, or when the prohibition sits far from the point of action. Vendor prompting guidance (e.g. Anthropic's) still recommends telling the model what to do rather than what not to do.

## Why it happens

Two causes are direct and observable:
- "Don't do X" leaves the correct action undefined. The model fills the gap with its default behavior, which may be exactly X.
- In long contexts, "don't do X" stated once early is followed less reliably by the time the model reaches the decision point. A positive instruction ("do Y") at the decision point is more effective.

Two further explanations are working hypotheses, not established mechanisms:
- Training text mostly describes what to do, so positive instructions map onto more familiar patterns.
- A prohibition puts the forbidden action into context, which may make it more salient rather than less.

## Analogy

Tell yourself "don't think about a pink elephant." You just thought about one. Now try "picture a blue square" — that works immediately. The brain latches onto the concrete concept in the instruction. Negative instructions likely work the same way on a model — they put the forbidden action in front of it right before asking it not to act on it. Positive instructions hand it the action to take instead.

## Symptoms

- Skill includes "don't generate code without asking first" — model generates code without asking
- Skill says "never activate for conceptual questions" — model activates for conceptual questions
- "Don't blend platform formats" — model blends formats anyway when user's request is ambiguous
- Model follows positive rules in a skill reliably but violates the negative ones
- "NEVER" or "ABSOLUTELY DO NOT" in caps gets added to force compliance — on recent Claude models (per Anthropic's prompting guidance) this tends to overcorrect, applying the rule too broadly (e.g. refusing adjacent, legitimate actions). Calm, specific phrasing with the reason attached works better than volume

## Fix

**Reframe every negative as a positive:**

| Negative (weaker) | Positive (stronger) |
|-----------------------|---------------------|
| Don't generate without confirmation | Ask for confirmation before generating |
| Never blend platform formats | Generate one file per platform, clearly labeled |
| Avoid activating for conceptual questions | Activate only when code modification is requested |
| Don't add rules the user didn't ask for | Add only rules explicitly requested by the user |
| Never skip negative triggers | Define negative triggers before writing the description |

**Structural fixes:**
- Audit every instruction in your skill for "don't", "never", "avoid", "no", "without". Rewrite each one as what to do instead.
- If you can't rewrite a negative as a positive, the constraint is probably unclear. Clarify what the desired behavior actually is.
- When a boundary truly must be stated negatively (out-of-scope declarations), keep it in a dedicated `## Boundaries` section and follow each negative with a redirect: "For X, say 'out of scope' and suggest Y instead."

**At decision points:**
- Place the positive instruction immediately before the action it governs, not only at the top of the file.
- Use conditional positive framing: "If the user asks about X, do Y" rather than "If the user asks about X, don't do Z."

## Example

**Bad — negation-heavy:**

```markdown
## Rules
- Don't generate without knowing the target platform
- Never use Claude Code tool names in a Cursor skill
- Don't activate if the user is just asking a question
- Never produce output longer than 500 lines
- Don't ask more than 2 clarifying questions
```

**Good — positive reframe:**

```markdown
## Rules
- Confirm target platform before generating any output
- Use intent-based language ("search the codebase") — never platform tool names
- Activate only when the user is requesting skill creation or modification
- Keep output under 500 lines; move excess to references/
- Ask at most 2 clarifying questions, then generate with stated assumptions
```

Same constraints. The positive version gives the model a clear action to take at each decision point.
