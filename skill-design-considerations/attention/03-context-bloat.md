# Context Bloat

## What it is

Context bloat occurs when a skill loads more content into the context window than is needed for the current step of the task. This crowds out working memory, pushes important content toward the middle of the context (worsening lost-in-the-middle), increases cost, slows response time, and — past a threshold — actively degrades instruction-following quality.

Bloat typically comes from:
- SKILL.md files that contain all detail inline instead of in references/
- Loading all reference files upfront instead of on demand
- Skills that include exhaustive examples, templates, and edge-case documentation directly in the body
- Long asset templates embedded in the skill rather than in assets/
- Multiple skills loaded simultaneously with overlapping content

## Why it happens

Skill authors are incentivized to be thorough — the more detail in the skill, the more confident they are the model has what it needs. This intuition is wrong above a certain threshold. Past roughly 2,000–4,000 tokens of instructions, additional content begins to compete with and displace earlier content rather than supplement it.

The mechanism:
- Context windows have fixed capacity. Every token of bloated instructions is a token not available for the user's actual request, conversation history, and working memory during generation.
- Models don't read instructions like humans read documentation. They can't "skip to the relevant part." Every token in context influences every generated token, creating noise.
- The attention mechanism dilutes as context grows. A critical instruction in a 500-token skill gets more reliable attention than the same instruction in a 5,000-token skill.

## Analogy

Imagine packing every item you own into one suitcase "just in case." At the airport you can't find your passport — it's buried under three pairs of shoes you never needed. More stuff doesn't help; it actively buries what matters. A skill that loads all its content into context does the same thing: the critical instructions are in there, but so is everything else, and the passport is lost somewhere in the middle.

## Symptoms

- SKILL.md file is 300+ lines with inline examples, full templates, and exhaustive edge cases
- Model loads skill and then ignores half the instructions it just loaded
- Skill works well for simple cases but degrades for complex ones (complex cases push context further)
- Instruction-following gets worse as conversation grows longer (bloat + conversation history fills the window)
- Skill includes the same information in multiple places "for emphasis" — actually just doubles the noise
- Cost per skill invocation is noticeably high

## Fix

**Progressive disclosure architecture:**

This is the core fix. Organize skill content into tiers based on when it's actually needed:

```
Tier 0 — Always loaded (frontmatter name + description only): ~100 tokens per skill
  Trigger matching — every installed skill pays this cost in every session

Tier 1 — Loaded on activation (SKILL.md body): 50–150 lines
  Core workflow, top-level constraints, references to tier 2

Tier 2 — Loaded on demand (references/): loaded when a specific subtask requires it
  Full platform specs, detailed procedures, edge-case documentation

Tier 3 — Not loaded into context (assets/): used by the model as templates to copy
  Boilerplate files, starter templates, full examples
```

Only tier 0 is resident before the skill fires; the SKILL.md body enters context when the agent decides to activate the skill. The model loads tier 2 only when it determines it needs the detail. It copies from tier 3 without the template text entering context.

**Lines as a token proxy:**

Line counts in this guide are proxies — the model processes tokens, not lines, and the ratio varies. Rough averages for typical SKILL.md content:

- Prose: ~15–25 tokens/line
- Code: ~5–15 tokens/line (highly variable by language)
- Markdown structural lines (headers, short bullets): ~3–10 tokens/line
- Typical mixed SKILL.md: ~8–15 tokens/line

A 150-line SKILL.md of mixed content is roughly 1,000–2,500 tokens. For precision, measure tokens directly with a tokenizer rather than relying on line counts.

**SKILL.md size targets:**

| Type | Target (lines) | Approximate tokens |
|------|----------------|--------------------|
| Simple one-task skill | 30–80 lines | ~300–800 tokens |
| Multi-step workflow skill | 80–150 lines | ~800–2,000 tokens |
| Complex orchestrator skill | 150–250 lines | ~2,000–4,000 tokens |

These targets are this guide's house style, deliberately stricter than the standard. The Agent Skills spec sets the outer bound: keep SKILL.md under 500 lines and under ~5,000 tokens. Past 250 lines, treat it as a signal to move content to references/; past 500, it's out of spec.

**What belongs in SKILL.md vs references/:**

| SKILL.md | references/ |
|----------|-------------|
| Trigger conditions | Full platform format specs |
| High-level workflow steps | Detailed step-by-step procedures |
| Top-level constraints (5 max) | Edge cases and exceptions |
| Pointers to references/ | Full examples and templates |
| Completion signal | Troubleshooting guides |

**Avoid inline examples:**

Examples are the biggest source of bloat. A single well-written example can be 50+ lines. Instead of embedding examples:

```markdown
## Example
See `references/examples.md` for before/after patterns.
```

Load the example only if the user asks for one or the model determines it needs one.

**Avoid redundancy:**

"For emphasis" repetition doubles bloat without doubling attention. If a constraint needs to appear twice, use a one-line bold reminder instead of repeating the full statement.

**Test context size:**

Before publishing a skill, measure its loaded size. For Claude Code:
- Check token count of SKILL.md + any always-loaded references
- Target under 1,500 tokens for the always-loaded tier (≈ 75–150 lines of mixed prose/code)
- Verify that the skill + a typical user conversation fits comfortably within the context window

## Example

**Bad — everything inline, 400+ lines:**

```markdown
---
name: skill-creator
description: Creates skills for Claude Code and Cursor.
---

## On Invoke
...

## Workflow
...50 lines of detailed procedures...

## Platform: Claude Code
...full format spec inline (80 lines)...

## Platform: Cursor
...full format spec inline (80 lines)...

## Examples
### Good Claude Code skill
...50-line full example...

### Good Cursor rule
...50-line full example...

## Edge Cases
...40 lines of edge cases...

## Templates
...full boilerplate templates inline...
```

All 400 lines load every time, even if the user only needs Claude Code format.

**Good — progressive disclosure, 80-line SKILL.md:**

```markdown
---
name: skill-creator
description: >
  Generates SKILL.md files and platform rules for Claude Code, Cursor,
  Windsurf, Copilot, Aider, OpenCode. Triggers on skill creation requests.
---

## On Invoke
Confirm target platform. Load the relevant platform reference.

## Workflow
1. Confirm platform (or ask)
2. Load `references/<platform>.md`
3. Extract requirements from user
4. Generate output per platform spec
5. Confirm with user

## Rules
1. Load platform reference before generating — do not rely on memory
2. Include negative triggers in all output
3. One file per platform

Full specs: `references/claude-code.md`, `references/cursor.md`, etc.
Examples: `references/examples.md` (load only if needed)
Templates: `assets/<platform>-template.md` (copy, do not load into context)

## Completion
Confirm output with user. Exit skill-creation mode.
```

80 lines always loaded. Detail is pulled from references/ only when needed for the specific platform being targeted.
