# Lost in the Middle

## What it is

LLM attention is not uniform across a long context. Models pay the most attention to content at the **start** and **end** of their input. Content placed in the middle of a long prompt — even critical rules or constraints — receives significantly less attention and is more likely to be ignored or forgotten during generation.

This applies at multiple scales:
- Within a single SKILL.md file (rules buried on line 200 of 400)
- Within a list of instructions (item 6 of 10)
- Across the full context window (skill loaded in the middle of a long conversation)

## Why it happens

Transformer attention is theoretically global, but in practice trained models exhibit a U-shaped recall curve when tested on long inputs. The phenomenon was documented empirically in "Lost in the Middle: How Language Models Use Long Contexts" (Liu et al., 2023). That study tested 2023-era models; newer long-context models do much better on simple needle-retrieval tests. What persists is broader degradation as context grows: "Context Rot: How Increasing Input Tokens Impacts LLM Performance" (Chroma, 2025) tested 18 models including Claude 4, GPT-4.1, and Gemini 2.5 and found performance becomes less reliable with every increase in input length — well before the context limit — and worsens with distractors and semantically similar content. Treat the U-curve as a directional risk, not a precise map.

Likely causes:
- Training data rarely requires retrieving facts from the exact middle of long documents
- Positional encoding interacts with attention in ways that favor endpoints
- RLHF and instruction tuning reinforce following instructions in familiar positions (beginning of system prompt, beginning of user turn)

## List-Level Effects (Primacy-Recency Bias)

This section extrapolates the long-context finding down to short lists. It is a design heuristic, not a measured result — but it's cheap to follow and matches how humans write priorities.

The same U-shaped pattern is assumed to operate at micro-scale within individual lists:

**Primacy bias** — the first item in a list is processed when the model's attention is fully available and the context is least cluttered. It anchors how the rest of the list is interpreted.

**Recency bias** — the last item is closest to the point of generation. It's the most recently "seen" instruction and most likely to influence the immediately following output.

Items 2 through N-1 in a list compete with each other in the attention-poor middle zone. Both effects are amplified by training data patterns: instructions tend to state the most important thing first (human writing convention), so models learn to weight early items more heavily.

This means for any list of rules, workflow steps, or requirements:
- 3-item lists have no middle zone
- 5-item lists have a small one
- 10-item lists have a large middle zone where items are more likely to be skipped

## Analogy

Think of a keynote speech. The audience remembers the speaker's opening joke and their closing call-to-action. The eight bullet points in the middle? Gone by the next morning. Your SKILL.md is that speech — the crowd (the model) was present for all of it, but only the edges survived.

Same for a grocery list: you remember milk (first thing written, before the list got long) and eggs (added five minutes ago). The six items in the middle? You're standing in the store trying to recall them. Instruction lists work the same way at every scale.

## Symptoms

**Document-level:**
- Model follows rules stated at the top of SKILL.md but ignores rules halfway down
- A constraint mentioned only once in the middle of a long skill body gets violated
- Skill works correctly on short conversations, breaks on long ones (middle content pushed further from edges as context grows)
- Model completes the wrong step in a multi-step workflow when the correct step was listed in the middle
- User corrects the model for ignoring something "clearly stated" — and it was in the middle of the file

**List-level:**
- Rule #1 and rule #8 in a list are followed; rules #3–#6 are frequently violated
- The last step in a workflow is executed correctly; middle steps are skipped or reordered
- Adding a rule at the end of a list temporarily "fixes" a compliance issue; adding it in the middle doesn't
- Multi-item instructions produce outputs that match the first and last items but miss middle requirements

## Fix

**Structural fixes:**
- Keep SKILL.md under 150 lines for the most critical path. Push detail to `references/`. (500 lines is Anthropic's recommended SKILL.md ceiling — a guideline, not an enforced cap; 150 is the attention-quality target, well inside it, because recall degrades long before you approach the recommended limit.)
- State the single most important constraint in the first 5 lines of the skill body.
- Repeat critical constraints at the end of the file as a summary or checklist.
- Use progressive disclosure: load only what's needed for the current step.

**Ordering fixes:**
- Put the most important rule first in every list, not last.
- If you have 10 rules, cut to 5. The ones you cut were already lost in the middle.
- Never bury a negative trigger (when NOT to activate) in the middle — put it immediately after the description.

**Content fixes:**
- Use section headers to create visual anchors. Models attend to headers even in middle content.
- Bold critical constraints: `**Never generate output without user confirmation.**`
- For multi-step workflows, number steps and keep each step's instruction on one line.

**List-level fixes:**
- Structure lists with primacy and recency in mind: item 1 = most critical constraint; items 2 to N-1 = supporting details; item N = second most critical, or restatement of item 1.
- Target 3–5 items per list. If you need more, split into separate sections — each section's first and last items get full attention.
- Use section headers to create multiple "edges" that reset the primacy/recency dynamic:

```markdown
## Step 1: Requirements
1. Extract objective (primacy — gets full attention)
2. Identify triggers
3. Confirm negative triggers (recency — gets good attention)

## Step 2: Platform
1. Detect from context (primacy)
2. Ask if ambiguous (recency)
```

6 instructions split across 2 sections, each with its own primacy/recency pair. Much more reliable than a flat list of 6.

## Example

**Bad — critical constraint buried mid-file:**

```markdown
## On Invoke
Check the user's goal.

## Workflow
1. Gather requirements
2. Detect platform
3. Generate skill body
4. Always define negative triggers before writing the description field — this is required.
5. Format output
6. Review

## Rules
...15 more rules...
```

Step 4's constraint is lost. It will be ignored.

**Good — critical constraint surfaced immediately:**

```markdown
**Required before generating:** negative triggers must be defined. Ask if missing.

## On Invoke
Check the user's goal. Confirm negative triggers exist before proceeding.

## Workflow
1. Gather requirements — confirm negative triggers defined
2. Detect platform
3. Generate skill body
4. Format output

## Rules
...5 rules max...

**Reminder:** do not generate without negative triggers confirmed.
```

Same constraint appears at top, embedded in the workflow step, and restated at the end.

**Bad — critical constraint in the middle of a rule list:**

```markdown
## Rules
1. Use kebab-case for file names
2. Keep SKILL.md under 500 lines
3. Always include negative triggers — this is the most important rule
4. Use intent-based language
5. One file per platform
```

Rule 3 ("most important rule") is in position 3 of 5. It's in the middle zone. It will be violated more than rules 1 and 5.

**Good — critical constraint at list edges:**

```markdown
## Rules
1. Always include negative triggers in every generated skill
2. Keep SKILL.md under 500 lines; move excess to references/
3. Use intent-based language ("search the codebase", not tool names)
4. One file per platform; use correct file names per platform conventions

**Required in all outputs: negative triggers must be defined.**
```

Most important rule first. Restated as bold reminder at end. Middle rules are important but survivable if occasionally missed.
