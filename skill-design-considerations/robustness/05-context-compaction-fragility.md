# Context Compaction Fragility

## What it is

When a conversation grows long, platforms compress or summarize earlier turns to free context space. This compression is lossy — details are dropped, decisions are paraphrased, and specific values are generalized. A skill that relies on information established in earlier turns (platform choice, user requirements, intermediate decisions) loses that information when compaction occurs. The skill's workflow continues, but on a foundation that's been partially erased.

This is distinct from conversational drift (grounding/04), where instructions are gradually overridden by recent conversation. Compaction fragility is about literal information loss — the content is removed from context, not just deprioritized.

## Why it happens

Platforms compact context to stay within the model's context window. The compaction algorithm summarizes earlier turns, preserving what it estimates is most important. But the algorithm doesn't know which details the skill needs — it optimizes for general conversation coherence, not for skill state preservation.

A skill that established "target platform: Cursor" in turn 3 may find that detail summarized away by turn 30. The compacted context says "discussed skill creation requirements" — accurate but missing the specific platform decision. The model either re-asks (annoying) or guesses (risky).

Multi-step workflows are most vulnerable because they accumulate state over many turns — exactly the turns most likely to be compacted.

## Analogy

Meeting notes taken by a summarizer who wasn't told which decisions matter. The full transcript from the morning session recorded that the team chose PostgreSQL over MySQL, with specific reasons. The afternoon summary says "database technology was discussed and a decision was reached." The decision was made. Nobody knows what it was. The team either re-decides (wasting time) or someone guesses wrong.

## Symptoms

- Skill re-asks questions the user already answered earlier in the conversation
- Output in later steps contradicts decisions made in earlier steps (the decision was compacted away)
- Multi-step workflow produces inconsistent results across steps — each step generates fresh assumptions
- Skill works correctly in short conversations but degrades after the first compaction — compaction triggers on token usage, not turn count, so a few turns with large tool outputs can trigger it as easily as many short ones
- User says "I already told you that" — because they did, but the information was compacted

## Fix

**Repeat key decisions at each step:**

Don't rely on earlier turns surviving compaction. Restate important state at the point where it's used:

```markdown
## Workflow
1. Gather requirements — confirm platform, name, triggers
2. Generate draft
   Before generating, restate: "Generating for [platform], skill name: [name]"
3. Revise if needed — restate what's being revised and why
```

**Write state to memory when available:**

If the platform supports persistent memory, save key decisions there rather than relying on conversation context:

```markdown
## State Persistence
After confirming requirements: save to memory:
- Target platform
- Skill name
- Core requirements summary

Before each subsequent step: verify state from memory, not conversation history.
```

**Make each step self-contained:**

Design workflow steps so each can execute correctly even if all previous turns were summarized to one line:

```markdown
## Step 3: Generate
Inputs needed for this step (re-verify before proceeding):
- Platform: [must be confirmed, not assumed from earlier]
- Requirements: [list the specific requirements this step acts on]
- Constraints: [restate constraints that affect generation]

If any input is missing or unclear: ask before generating.
```

**Embed state in structured output:**

When the skill produces intermediate output (drafts, plans, partial results), embed the decisions that led to it:

```markdown
## Output Format for Intermediate Steps
Include a state header in each intermediate output:

**Current state:**
- Platform: Cursor
- Phase: Generation (step 2 of 4)
- Requirements confirmed: [list]
- Open questions: [list or "none"]
```

This survives compaction because it's in the model's own output, not in a user turn that might be summarized.

**Design for recovery, not just prevention:**

Accept that compaction will happen. Define how the skill recovers:

```markdown
## On State Loss
If a required piece of information seems to be missing from context:
1. Check memory (if available)
2. State what you remember and what's uncertain
3. Ask the user to confirm: "I recall we chose Cursor — is that right?"

One-line confirmation is cheaper than regenerating from wrong assumptions.
```

## Example

**Bad — relies on earlier turns surviving:**

```markdown
## Workflow
1. Ask for platform and requirements
2. Generate the skill file based on the answers from step 1
3. Ask for feedback
4. Apply revisions based on all prior context
```

By step 4, the "answers from step 1" may be compacted. The model either re-asks everything or guesses based on whatever fragments survive in the summary.

**Good — state restated and persisted:**

```markdown
## Workflow
1. Gather: platform, skill name, requirements, negative triggers
   → Save confirmed state to memory (if available)
   → Summarize confirmed state in response: "Confirmed: [platform], [name], [requirements]"

2. Generate — before generating, restate:
   "Generating [name] for [platform] with requirements: [list]"
   If any detail is uncertain, ask before proceeding.

3. Review — present output with state header:
   "**Skill:** [name] | **Platform:** [platform] | **Status:** awaiting review"

4. Revise — restate what's changing and what's preserved:
   "Keeping: [unchanged elements]. Changing: [specific revisions]."

## Compaction Safety
Every step restates the state it depends on.
No step relies solely on information from a previous turn.
```

Same workflow. Each step carries its own context. Compaction doesn't break the chain.
