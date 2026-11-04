# Lesson 2: Agent Basics

## Concept

An ADK `Agent` (an `LlmAgent` under the hood) is defined by a handful of fields that all matter:
- `name` — a stable identifier, used when agents reference each other later (Lesson 7).
- `model` — which LLM backs this agent.
- `description` — a short summary another agent or the platform uses to decide when to delegate to this agent. Not shown to the end user.
- `instruction` — the system prompt: the actual behavioral contract for this agent. This is what you'll iterate on most.
- `tools` — a list of callables/tool objects the agent can invoke (empty for now, added in Lesson 3).

**Description vs. instruction** is the detail learners most often blur: description is *metadata for orchestration* (who am I, when should I be picked), instruction is *behavior* (how should I act once picked). Getting this backwards makes multi-agent delegation in Lesson 7 confusing later.

## Task

Replace the Lesson 1 placeholder with the real project skeleton:
1. Rename the agent to something like `support_agent`.
2. Write a `description` that states its role in one sentence (e.g. "Answers customer questions about [your product]").
3. Write an `instruction` that sets tone and boundaries (e.g. "You are a support agent for X. Be concise. If you don't know something, say so — do not make up answers.").
4. Run it via `adk run` with 2-3 realistic support questions and observe how the instruction shapes the responses.

## Acceptance Criteria

- `name`, `description`, and `instruction` are all filled in with real, specific content (not placeholders).
- The learner can articulate, in their own words, the difference between what `description` and `instruction` each do.
- Test responses via `adk run` visibly follow the instruction's tone/boundary rules.

## Common Mistakes

- Writing a vague instruction ("be helpful") that doesn't actually constrain behavior — push for something testable (a tone, a boundary, a refusal condition).
- Putting orchestration metadata (who should route to me) inside `instruction` instead of `description`.
- Skipping the "why does the instruction matter" check by not actually running test questions before moving on.
