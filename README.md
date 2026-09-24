# skillanomics

A personalized skill registry for coding agents — reusable agent skills designed for Claude Code, with portability to Cursor, Windsurf, Copilot, Aider, and OpenCode.

## Skills

| Skill | Description |
|-------|-------------|
| `skill-tutor` | Teaches how to build optimal, portable agent skills for coding agents |
| `skill-creator-agnostic` | Generates SKILL.md skills (Agent Skills standard) and native rules files for any coding agent platform |
| `learn-react` | Guided React + Vite learning assistant — teaches by doing with real project goals and code review |
| `learn-typescript` | Guided TypeScript learning assistant — type system fundamentals to advanced types, with every workshop verified by the compiler |
| `learn-nextjs` | Guided Next.js App Router + TypeScript learning assistant — builds on learn-react foundations |
| `learn-python` | Python learning skill |
| `web-automation` | Writes Python Playwright scripts for browser automation — forms, auth flows, pagination, multi-tab |
| `sdet-test-auditor` | Audits test suites for quality and coverage gaps |
| `sdet-test-builder` | Generates test scaffolding and test cases |

## Design Considerations

`skill-design-considerations/` is a field guide to the failure modes that break agent skills. Organized into 9 categories:

- **Attention** — how models allocate focus and why critical instructions get ignored
- **Grounding** — keeping the model tethered to skill instructions as conversations evolve
- **Robustness** — designing for incomplete inputs, platform variance, and edge cases
- **Composition** — how skills behave alongside other skills and grow over time
- **Anatomy** — structural and architectural failure modes in how a skill file is assembled
- **Calibration** — how skill instructions shape model confidence and uncertainty
- **Interaction** — the UX contract between skill and user
- **Security** — handling untrusted data and elevated capabilities
- **Principles** — positive design guidance for what good skill design looks like

39 considerations in total, each with an analogy, symptoms, fix, and before/after example.

## Installation

### Project-scoped (recommended)

Copy a skill into your project's `.claude/skills/` directory:

```bash
cp -r skills/<skill-name>/ /path/to/your-project/.claude/skills/<skill-name>
```

The skill will only be active when working in that project.

### Global

Install a skill into your global skills directory:

```bash
# Install all skills
npx skills add pcphil/skillanomics

# Install a specific skill
npx skills add pcphil/skillanomics --skill skill-tutor

# Full GitHub URL
npx skills add https://github.com/pcphil/skillanomics

# Local path
npx skills add ./skills/<skill-name>
```

The skill will be available in all projects.

### Manual

Clone the repo and copy the skill directory to your coding agent's skills folder.

### Verify

After installing, run `/reload-plugins` in Claude Code, then check `/skills` to confirm it appears.
