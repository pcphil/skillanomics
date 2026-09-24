# Curriculum: Building Portable Agent Skills

## Module 1: Portability Patterns

**Goal:** Understand how to write skills that work across multiple coding agent platforms.

### Lessons

1.1 **What "Portable" Means**
- Separating intent from platform syntax
- The universal core: every agent reads instructions and has tools
- Why portability matters even if you only use one agent today

1.2 **The Agent Interface Landscape**
- Two file kinds: skills (`SKILL.md`, Agent Skills open standard) and rules/instruction files (per-tool, plus `AGENTS.md`)
- SKILL.md readers: Claude Code, GitHub Copilot, Cursor, Windsurf (Devin Desktop), OpenCode, and others
- Rules: `.cursor/rules/*.mdc`, `.windsurf/rules/*.md`, `.github/copilot-instructions.md` and `.github/instructions/*.instructions.md`, `CONVENTIONS.md` (Aider)
- What they share vs. where they diverge; verify against vendor docs, they change fast

1.3 **Universal Core, Thin Wrapper**
- Pattern: write the core once as a standard SKILL.md, adapt only where a tool lacks skills
- What belongs in the SKILL.md (name, description, workflow, references) vs. an adapter (condensed rules file, platform features with fallbacks)
- Example: a code review skill as SKILL.md plus a condensed CONVENTIONS.md for Aider

1.4 **Mapping Between Platforms**
- Translation table: SKILL.md concepts vs. rules-file equivalents
- Handling capabilities that don't exist everywhere (memory, file-scoped activation, extension frontmatter fields)
- Graceful degradation vs. platform-specific branches

1.5 **Anti-Patterns**
- Tight coupling to one agent's quirks
- Assuming specific tool names or invocation syntax
- Embedding platform paths in core logic
- Over-engineering portability for a single-platform skill

### Exercise
Take an existing Claude Code skill and list which frontmatter fields are standard and which are extensions. Write a 1-paragraph "portability brief" describing what it needs to run on Aider.

---

## Module 2: Platform-Native Formats

**Goal:** Master the Agent Skills standard and each platform's rules-file format and capabilities.

### Lessons

2.1 **Agent Skills Standard: SKILL.md**
- Required: `name` (max 64 chars, lowercase, hyphens, matches directory) and `description` (max 1024 chars)
- Optional standard fields: `license`, `compatibility`, `metadata`, `allowed-tools`
- Directory layout: SKILL.md + references/ + scripts/ + assets/
- Progressive disclosure: metadata at startup, body on activation, resources on demand
- Description drives trigger matching: write it as a discrimination signal, with negative triggers

2.2 **Claude Code Extensions**
- Extra frontmatter: `when_to_use`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `context: fork`, `paths`, `hooks`, `model`, `effort`, and more
- Extensions work in Claude Code; claude.ai uploads and strict standard validators accept only the standard fields
- Memory system for state persistence; slash-command invocation; string substitutions
- Description plus `when_to_use` capped at 1,536 characters

2.3 **Cursor: Skills and Rules**
- Skills in `.cursor/skills/` or `.agents/skills/`; `disable-model-invocation` turns a skill into a slash command
- Rules: `.cursor/rules/*.mdc` with `description`, `globs`, `alwaysApply`; four types (always, intelligent, file-scoped, manual)
- `AGENTS.md` as a plain alternative; rules under 500 lines

2.4 **Windsurf (Devin Desktop): Skills and Rules**
- Skills in `.devin/skills/`; rules in `.devin/rules/*.md` or `.windsurf/rules/*.md`
- Rule `trigger:` modes: `always_on`, `model_decision`, `glob`, `manual`; 12,000-character limit per rule file
- Use skills for multi-step work with supporting files; rules for short behavioral guidance

2.5 **GitHub Copilot: Skills and Instructions**
- Skills in `.github/skills/` (also reads `.claude/skills/`, `.agents/skills/`)
- Instructions: `.github/copilot-instructions.md`, path-scoped `.github/instructions/*.instructions.md` with `applyTo`, and `AGENTS.md`
- Keep instructions to about 2 pages

2.6 **Aider: Conventions**
- `CONVENTIONS.md` loaded with `--read` or `read:` in `.aider.conf.yml` (read-only, cacheable)
- Plain markdown; no skills support found in its docs; keep it lean

2.7 **OpenCode: SKILL.md**
- Same skill format; searches `.opencode/skills/`, `.claude/skills/`, `.agents/skills/`
- `name` must match the directory; permission system (`allow`/`deny`/`ask`) in `opencode.json`
- Use intent-based language (tool names differ)

2.8 **What's Truly Universal vs. Platform-Specific**
- Universal: `name`, `description`, workflow steps, constraints, boundaries, `references/`
- Platform-specific: Claude Code extension fields and memory, per-tool rule triggers (`globs`, `applyTo`, `trigger:`), permission systems
- When to use platform-specific features vs. when to stay on the standard

### Exercise
Create a skill for a platform you use. Then describe what would change to port it to a second platform.

---

## Module 3: Prompt Engineering for Skills

**Goal:** Write skill instructions that produce consistent, high-quality agent behavior.

### Lessons

3.1 **Imperative vs. Descriptive Instructions**
3.2 **Positive Framing — Why "Do X" Beats "Don't Y"**
3.3 **Examples as Specification (and the Anchoring Risk)**
3.4 **Handling Ambiguity and Edge Cases**
3.5 **Voice and Tone Calibration**

---

## Module 4: Tool & Resource Design

**Goal:** Know when and how to use scripts, references, assets, and progressive disclosure effectively.

### Lessons

4.1 **Scripts: Deterministic Reliability**
- When to use scripts vs. instructions
- Naming and location conventions

4.2 **References: On-Demand Knowledge**
- When to create references, naming conventions, progressive disclosure
- Structural archetypes: teaching, generator, utility
- Full reference: `references/anatomy.md`

4.3 **Assets: Templates and Boilerplate**
- When to use assets vs. references
- Template design for copy-and-modify workflows

4.4 **Context Budget Management**
- Platform size targets, SKILL.md vs. references allocation
- Decision framework for structural patterns (phases, modes, subcommands, state)
- Full reference: `references/decisions.md`

4.5 **Platform-Specific Features in Practice**
- Claude Code: progressive disclosure, memory, named tools
- Cursor: globs for auto-attach, context variables, alwaysApply
- When these features are worth using vs. when they harm portability

---

## Module 5: Skill Design Considerations

**Goal:** Recognize and prevent the failure modes that break skills in production.

### Lessons

5.1 **Attention Failures**
- Lost in the middle, over-specification, context bloat
- How to structure instructions for reliable attention

5.2 **Grounding Failures**
- Negation failure, state leakage, conversational drift, persona capture
- Building termination signals and scoped behavior

5.3 **Robustness and Composition**
- Trigger pollution, happy-path-only, platform assumptions
- Skill composition, scope creep, priority inversion

5.4 **Calibration and Interaction**
- Overconfidence, hallucination amplification, example anchoring
- Silent failure, mode opacity, feedback loops

5.5 **Adversarial Testing**
- Testing with missing input, wrong input type, out-of-scope requests
- Pre-ship checklist for skill quality

Full taxonomy: `../../skill-design-considerations/`
