# Skill Design Decisions

A framework for structural choices when designing skills. Each decision follows: **Question → Criteria → Example → Default.**

For the structural templates themselves (what sections exist, how to order them), see `anatomy.md`.

---

## Decision: Phases vs. Flat Workflow

**Question:** Does this skill need named phases with explicit transitions?

**Criteria — use phases when:**
1. The skill has 2+ distinct behavioral modes (gathering requirements is different from generating output)
2. User confirmation gates exist between stages (approval before proceeding)
3. A revision loop is needed (output → feedback → re-generation → re-review)

**Example:** `skill-creator-agnostic` uses 5 named phases because each requires fundamentally different behavior:
- `[Requirements]` — asking questions, confirming inputs
- `[Generating]` — loading references, producing output, running checklist
- `[Review]` — presenting output, waiting for approval
- `[Revising]` — applying targeted changes, re-running checklist
- `[Done]` — exiting skill mode

Phase transitions carry explicit messages: "Requirements confirmed: [platform], [name]. Generating now."

**Counter-example:** `learn-react` uses a flat Core Loop (Concept → Task → Wait → Review → Advance) because every cycle has the same behavior — there are no distinct modes or confirmation gates.

**Default:** Start flat. Add named phases only when you have at least two stages that behave differently and need a visible transition between them.

---

## Decision: References vs. Inline

**Question:** Should this content live in a reference file or stay in SKILL.md?

**Criteria — create a reference file when:**
1. **Size**: the content exceeds ~50 lines of detail
2. **Conditional loading**: the content isn't needed on every invocation (per-lesson, per-platform, per-mode)
3. **Independent updates**: the content changes on a different cadence than core skill logic

**Example:** `learn-python` keeps the curriculum overview inline (30-line table in SKILL.md) but puts each lesson's exercises, acceptance criteria, and teaching notes in its own reference file (`references/p1-l1-environment.md`). Only the current lesson loads at any time — 19 other files stay out of context.

`skill-creator-agnostic` keeps platform detection inline (10-line lookup table) but puts each platform's format specification in its own reference file (`references/claude-code.md`). Only the target platform's reference loads.

**Counter-example:** `learn-typescript` at 30 lines has no reference files — everything fits in SKILL.md. Splitting would add file management overhead with no context savings.

**Default:** If you're debating whether content belongs in a reference, make it a reference. The cost of an extra file is lower than the cost of context bloat. You can always inline later; removing bloat is harder.

---

## Decision: Subcommands vs. No Subcommands

**Question:** Does this skill need named entry points like `/skill-name next`?

**Criteria — add subcommands when:**
1. **Session persistence**: the skill runs across multiple messages with resumable progress
2. **Curriculum progression**: the user needs to advance through ordered steps
3. **Mode switching**: distinct behaviors are accessible by name (not just by description matching)

**Example:** `learn-python` has 5 subcommands because it has 20+ lessons across 3 phases:
```
/learn-python        — resume or start
/learn-python next   — advance to next lesson
/learn-python status — show current phase, lesson, what's been built
/learn-python track  — jump to a Phase 3 domain track
/learn-python stop   — save progress, end session
```

`skill-tutor` has subcommands for mode switching:
```
/skill-tutor critique  — enter Reviewer mode
/skill-tutor patterns  — enter Librarian mode
```

**Counter-example:** `skill-creator-agnostic` has no subcommands — its workflow is single-session (requirements → generate → review → done). Phase transitions happen naturally through conversation, not through explicit commands.

**Default:** Add subcommands only when the skill has state worth navigating. If the skill is one-shot (invoke → produce output → done), subcommands add complexity without value.

---

## Decision: Memory/State vs. Stateless

**Question:** Does this skill need to remember anything between sessions?

**Criteria — use memory when:**
1. **Progress tracking**: user's position in a curriculum or multi-session workflow needs persistence
2. **User context**: background, preferences, or assessment results affect behavior in future sessions
3. **Cumulative output**: the skill builds something over multiple sessions that would be lost without state

**Example:** `learn-python` saves to memory: current phase/lesson, completed lessons, user background, domain goal. On resume, it reads memory, summarizes where the user left off, and offers to continue. Without memory, every session would restart from scratch.

`learn-react` saves: background + goal + current concept. Lighter state, same pattern.

**Counter-example:** `skill-creator-agnostic` is stateless — each invocation gathers requirements fresh, generates output, gets approval, and is done. Nothing persists because nothing needs to.

**Portability note:** Memory is Claude Code-specific. On platforms without it (Cursor, Windsurf, Copilot, OpenCode, Aider), alternatives:
- **File-based state**: write progress to a file in the project (e.g., `.learning-progress.json`)
- **Accept statelessness**: design the skill to work without persistence (assessment at every session)

**Default:** Accept statelessness unless the skill genuinely spans multiple sessions. Most skills don't need memory — they run once and are done.

---

## Decision: Context Budget Allocation

**Question:** How should I split content between SKILL.md and references?

**Criteria:**
1. **Always-loaded content** (SKILL.md body) should be the minimum needed to operate correctly on any invocation
2. **On-demand content** (references/) should be everything else — detail that's only relevant during a specific workflow step
3. **Platform target size** constrains total SKILL.md length

**Budget framework:**

| Content type | Location | Example |
|-------------|----------|---------|
| Trigger matching, identity | Frontmatter | `name`, `description` |
| Role framing, critical constraint | First 5 lines of body | Opening line, hierarchy scope |
| Entry point, state check | On Invoke section | Check memory, branch on state |
| Core workflow steps | Body | Core Loop, Phases |
| Rules (max 5) | Body | Positive-framed constraints |
| Subcommands | Body | Navigation commands |
| Boundaries, completion signal | End of body | Redirects, exit trigger |
| Lesson details, exercises | references/ | Per-lesson files |
| Platform specs, schemas | references/ | Per-platform format guides |
| Detailed checklists | references/ | Review criteria, pattern library |
| Boilerplate to copy | assets/ | Blank templates |

**Example allocation:** `learn-python` puts 122 lines in SKILL.md (curriculum overview, core loop, rules) and 20 reference files for lesson detail. Each reference is loaded only during its lesson. If all lesson content were inline, SKILL.md would exceed 800 lines.

**Default:** Aim for SKILL.md under 150 lines. If you're over 200, move the largest section to a reference file. Common candidates: detailed checklists, per-item specifications, extended examples.

---

## Decision: Splitting vs. Monolithic

**Question:** Should this be one skill or two?

**Criteria — split when:**
1. **"And" in objective**: "teaches Python AND generates project scaffolds" — two skills with different triggers and workflows
2. **Unrelated trigger sets**: the skill would fire on completely different types of requests
3. **Size**: exceeding 500 lines even after moving content to references

**Example:** Skillanomics separated `skill-creator-agnostic` (generates skill files) from `skill-tutor` (teaches skill design). Both are about skills, but:
- Different triggers: "create a skill for X" vs. "teach me about skill design"
- Different workflows: phase-based generation vs. curriculum-based teaching
- Different outputs: skill files vs. knowledge transfer
- Different users: someone building a skill vs. someone learning to build skills

If combined, the skill would be ~210 lines with competing activation patterns and confused behavioral modes.

**Counter-example:** `skill-tutor` has three modes (Tutor, Reviewer, Librarian) in one skill because all three serve "skill design learning" and share references. Splitting would duplicate the reference loading and create confusing activation overlap.

**Default:** One objective = one skill. Test with: "This skill [verb]s [noun]." If you need "and" to describe it, consider splitting. But don't split modes that share context — that creates worse problems than a larger file.

---

## Decision: Modes vs. Separate Skills

**Question:** Should these behaviors be modes within one skill or separate skill files?

**Criteria — use modes when:**
1. **Shared context**: modes use the same domain knowledge, references, and terminology
2. **Natural transitions**: users switch between modes in the same session
3. **Trigger overlap**: the same description would reasonably match for all modes

**Criteria — use separate skills when:**
1. **Distinct references**: each behavior needs completely different supporting files
2. **Separate sessions**: users rarely switch between behaviors in one sitting
3. **Different trigger populations**: different user descriptions would match each behavior

**Example:** `skill-tutor` combines Tutor/Reviewer/Librarian because:
- All three share references (curriculum.md, patterns.md, critique.md, portability.md)
- A user might start learning (Tutor), then ask for a review of their draft (Reviewer), then look up a pattern (Librarian) — natural flow
- "Help me with skill design" could mean any of the three

Mode switching uses subcommands: `/skill-tutor critique`, `/skill-tutor patterns`

**Counter-example:** A "code review" skill and a "test generator" skill share a codebase but have completely different workflows, references, and trigger patterns. Combining them would force unrelated behaviors into one file with awkward mode switching.

**Default:** Modes when 70%+ of the context (references, terminology, domain knowledge) is shared across behaviors. Separate skills when behaviors have distinct identities that users would search for independently.

---

## Quick Reference

| Decision | Signal to add complexity | Default |
|----------|------------------------|---------|
| Phases | 2+ distinct behavioral modes | Start flat |
| References | >50 lines of conditional content | Make it a reference |
| Subcommands | Navigable persistent state | Skip |
| Memory | Multi-session progress | Accept stateless |
| Budget | SKILL.md > 200 lines | Move content out |
| Split | "And" in objective | One skill |
| Modes | 70%+ shared context | Modes over separate files |
