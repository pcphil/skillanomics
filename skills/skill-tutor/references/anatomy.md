# Skill Anatomy Reference

How to structure a skill file. Covers the standard sections, why they're ordered the way they are, and three structural archetypes drawn from real skills in this repo.

For the *decision* of when to use each structural element (phases, references, subcommands, modes), see `decisions.md`.

---

## Standard SKILL.md Sections

Every skill follows a subset of these sections. Not all are required — use only what the skill needs.

### 1. Frontmatter

```yaml
---
name: skill-name
description: >
  What this skill does, when it triggers, when it does NOT trigger.
---
```

**Purpose:** Identity and trigger matching. The `description` field is the primary discrimination signal — agents use it to decide whether to activate.

**Guidelines:**
- `name`: kebab-case, matches directory name
- `description`: include positive triggers ("teach me X", "create a Y") AND negative triggers ("does NOT activate for Z")
- Keep under 4 lines. Dense with keywords, not prose

### 2. Opening Line

```markdown
When teaching Python: reason as a structured learning tutor.
```

**Purpose:** Task-scoped role framing. Sets behavioral context for everything below.

**Guidelines:**
- Scope to the task: "When doing X: reason as Y" not "You are Y"
- Follow with hierarchy scoping: "These rules govern X only. Follow CLAUDE.md and system prompt for all other output."
- One sentence. Two max.

### 3. On Invoke

**Purpose:** What happens when the skill first activates. State check first (memory, files, context) if persistent state exists, then branch: resume path vs. fresh start path. Keep to 3-5 steps.

### 4. Requirements / Assessment

Teaching skills use **Assessment** (gather user background + goal). Generator skills use **Requirements** (gather objective, triggers, constraints).

**Purpose:** Collect the inputs needed before the skill can do its core work.

**Guidelines:**
- Ask all questions at once when possible (AskUserQuestion supports multi-question)
- Save answers to memory or state before proceeding
- Define what each answer controls downstream

### 5. Core Workflow

The heart of the skill. Three patterns:

- **Core Loop** (teaching): Concept → Task → Wait → Review → Advance. Repeats per lesson.
- **Phases** (generator): [Requirements] → [Generating] → [Review] → [Revising] → [Done]. Linear with backtracking.
- **Flat Workflow** (utility): numbered steps, no loops or phases.

**Guidelines:**
- Most critical step first AND referenced last (bookend rule)
- Each step is a concrete action, not a vague directive
- Include wait points where user input is needed

### 6. Rules / Constraints

```markdown
### Rules

1. One concept at a time. Never introduce two ideas at once.
2. Every task builds the real project — no throwaway exercises.
3. Read the user's actual file before giving feedback.
```

**Purpose:** Hard behavioral constraints the agent must follow.

**Guidelines:**
- Cap at 5 rules
- Positive framing: "do X" not "don't Y"
- Each rule must be actionable and verifiable

### 7. Subcommands

**Purpose:** Named entry points for navigating skill state. Format: `/skill-name action` (e.g., `/learn-python next`). Only add when the skill has navigable state (see `decisions.md > Subcommands`). Include a `stop` command for any skill that tracks progress.

### 8. Boundaries

**Purpose:** Where the skill stops. Explicit "not my job" declarations with redirect instructions. Be specific ("debugging existing code" not "unrelated tasks"). Include what the user should do instead. Place at end — boundaries are consulted when uncertain, not on every turn.

### 9. On Complete / Completion Signal

**Purpose:** Explicit exit. Define the trigger (user says "done", workflow completes, or topic changes), what to persist, and what to say on exit. Without this, the agent doesn't know when to stop being this skill.

---

## Section Ordering Rationale

The order above isn't arbitrary. Three principles drive it:

**Primacy bias.** Agents weight early content more heavily. Put the most critical constraint — the one thing that must never be violated — in the first 5 lines of the body, right after frontmatter.

**Workflow before rules.** The agent needs to know *what to do* before *what not to do*. Rules refine behavior; they don't define it. A rule without workflow context is ambiguous.

**Bookend rule.** Restate the single most critical constraint at the very end of the file. In long conversations, the agent's attention shifts toward recent content. Restating critical rules at the end counteracts attention decay.

**Boundaries last.** Boundary checking is a fallback — the agent consults boundaries when uncertain, not on every turn. Placing them early wastes attention on the happy path.

---

## Archetype 1: Teaching Skill

**Pattern:** Assessment → Core Loop → Curriculum → Subcommands → Memory

**Real example:** `skills/learn-python/SKILL.md` (122 lines)

```
Frontmatter (11 lines) — name, description with positive + negative triggers
Opening line (1 line) — scoped to Python learning
On Invoke (4 lines) — check memory → resume or assess
Assessment (15 lines) — background + domain goal via AskUserQuestion
Core Loop (10 lines) — Concept → Task → Wait → Review → Advance
Rules (5 lines) — one concept, real project, read actual file, guide not solve, adapt pace
Curriculum (30 lines) — inline table of concepts per phase
Reference Loading (6 lines) — load p{phase}-l{lesson}-{slug}.md one at a time
Subcommands (6 lines) — resume, next, status, track, stop
Pacing (4 lines) — stuck detection, tangent handling
Boundaries (5 lines) — scope limits with redirects
```

**Key structural features:**
- 20 reference files, each a single lesson — loaded one at a time during that lesson
- Reference naming: `p1-l1-environment.md`, `p1-l2-variables.md` — coded slugs for programmatic loading
- Curriculum inline in SKILL.md as a table — compact overview, details in reference files
- Subcommands for navigation: `/learn-python next`, `/learn-python status`
- Memory for progress: current phase/lesson, what they built

**Contrast:** `skills/learn-react/SKILL.md` (105 lines) uses the same archetype but with a single `references/curriculum.md` instead of per-lesson files. Simpler reference structure, fewer lessons, similar core loop.

---

## Archetype 2: Generator / Meta Skill

**Pattern:** Requirements → Platform Detection → Phases → Checklist → Blocking Conditions

**Real example:** `skills/skill-creator-agnostic/SKILL.md` (117 lines)

```
Frontmatter (4 lines) — name, description
Opening line (2 lines) — task-scoped role + hierarchy scoping
On Invoke (6 lines) — extract requirements, detect platform, load reference, generate
Requirements (8 lines) — objective, triggers, negative triggers, workflow, constraints
Platform Detection (10 lines) — signal → platform lookup table
Platform References (10 lines) — load target reference before generating
Output Rules (5 lines) — one platform per file, intent-based language, structure rules
Phases (10 lines) — [Requirements] → [Generating] → [Review] → [Revising] → [Done]
Blocking Conditions (4 lines) — platform unclear, requirements vague, premise wrong
Before Output Checklist (20 lines) — 7-discipline review covering all design considerations
Closing Reminder (1 line) — bookend of critical constraints
```

**Key structural features:**
- Named phases with mode indicators: `[Requirements]`, `[Generating]`, `[Review]`
- State transitions with explicit messages: "Requirements confirmed: [platform], [name]. Generating now."
- Revision loop: [Review] → [Revising] → back to [Review]
- Before Output checklist — a self-verification gate before presenting results
- 6 platform reference files — one per target platform, loaded based on detection
- Blocking conditions — explicit failure messages for the 3 most likely blockers
- No memory, no subcommands — single-session workflow

---

## Archetype 3: Simple Utility Skill

**Pattern:** Flat workflow, no state, minimal sections

**Real example:** `skills/learn-typescript/SKILL.md` (30 lines)

```
Frontmatter (4 lines) — name, description
Opening line (2 lines) — role statement
Modes (3 lines) — teach, practice, review
Lesson Flow (2 lines) — concept → example → checkpoint → hands-on → validate
Progress (3 lines) — memory tracking
Teaching Principles (5 lines) — lead with why, be direct, one concept, connect concepts
```

**What this demonstrates:**
- Minimum viable skill — enough to activate and guide basic behavior
- No On Invoke, no Assessment, no Subcommands, no Boundaries
- Short enough to stay in SKILL.md without references

**What it lacks:** Assessment, reference files, subcommands, boundaries, blocking conditions. Each would be added as complexity grows (see `decisions.md` for when each earns its place).

This archetype works for single-purpose utilities: a linter config skill, a commit formatter, a code review checklist. One thing, no state, under 50 lines.

---

## Reference File Anatomy

Reference files live in `references/` and are loaded on demand — not on every invocation.

### When to Create a Reference File

- Content exceeds ~50 lines of detail that isn't always needed
- Content is conditionally loaded (per-lesson, per-platform, per-mode)
- Content changes independently of core skill logic

### Naming Conventions

Two patterns in use:

**Descriptive slugs** — for topical references loaded by subject:
```
references/patterns.md
references/critique.md
references/portability.md
references/claude-code.md
```
Used by: skill-tutor, skill-creator-agnostic

**Coded slugs** — for sequential references loaded by position:
```
references/p1-l1-environment.md
references/p1-l2-variables.md
references/p3-web.md
```
Used by: learn-python (20 files across 3 phases + 4 tracks)

**Choose descriptive** when references are looked up by topic (librarian pattern).
**Choose coded** when references are loaded sequentially (curriculum pattern).

### Sizing

Individual reference files: under 300 lines. If a reference grows past 300, split it — the content is too dense for a single context load.

### Progressive Disclosure

Three tiers (see `patterns.md > Progressive Disclosure` for rationale):

1. **Always loaded:** SKILL.md frontmatter (~100 words) — paid every conversation
2. **On trigger:** SKILL.md body (<500 lines) — paid when skill activates
3. **On demand:** references/ files — paid only when explicitly loaded by a workflow step

State which reference to load at each workflow step. Example from learn-python: "Load only the reference file for that lesson: `references/p{phase}-l{lesson}-{slug}.md`. Do not load other lesson files."

---

## Size Budgets

### SKILL.md Body by Platform

| Platform | Target | Hard Max | Notes |
|----------|--------|----------|-------|
| SKILL.md (Claude Code, Cursor, Copilot, Windsurf, OpenCode) | 150 lines | 500 lines | Agent Skills spec; instructions under about 5,000 tokens; progressive disclosure offloads detail |
| Cursor rules (`.mdc`) | 150 lines | 500 lines | Per-rule, from Cursor docs |
| Windsurf rules | 100 lines | 12,000 characters per file | Enforced limit in Devin/Windsurf docs |
| Copilot instructions | 80 lines | about 2 pages | GitHub docs guidance |
| Aider conventions | 100 lines | no documented limit | Loaded read-only and cached; keep lean |

### Reference Files

- Individual file: under 300 lines
- Total reference count: no hard limit, but each load costs context
- Load one at a time — never bulk-load all references

### Total Footprint Guidance

A well-structured skill typically has:
- SKILL.md: 80–150 lines
- 0–6 reference files at 50–200 lines each
- 0–3 script files for deterministic operations
- 0–2 asset templates for boilerplate

If your SKILL.md exceeds 200 lines, move content to references. If you have more than 10 reference files, consider whether some can be merged or whether the skill should split.
