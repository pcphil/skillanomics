# Lesson 7: Multi-Agent & Eval (Required)

This lesson is required, not optional — do not let a learner skip to the capstone without it. Multi-agent composition and evaluation are what distinguish ADK from a single bare LLM call with tools.

## Concept

**Multi-agent orchestration**: a single agent handling every kind of support question eventually gets an overloaded instruction. ADK lets you compose agents — a root agent that delegates to specialized sub-agents based on their `description` (this is why Lesson 2 stressed writing a real description, not a placeholder). A common pattern here: a triage/escalation agent that decides whether a question needs the FAQ agent or should be flagged as "needs a human," each as its own `Agent` with a narrow, well-described role.

**`adk eval`**: rather than manually re-testing the same few questions after every change, ADK supports evaluation sets — a file of example inputs and expected-behavior criteria that can be run automatically to check whether the agent still behaves as intended. This is the mechanism that turns "I tried a few questions and it seemed fine" into a repeatable check.

## Task

1. Add a second agent — e.g. an `escalation_agent` with a narrow description ("Handles questions the FAQ agent can't answer, and tells the user a human will follow up") — and compose it with the support agent (e.g. via `sub_agents`, so the root agent can delegate to it).
2. Test that a question clearly outside the FAQ set (e.g. "why was I charged twice last month?") gets routed to the escalation agent, while an FAQ-covered question still goes to the original path.
3. Create a small `adk eval` evaluation set covering at least 3 cases: one clear FAQ hit, one clear escalation case, and one edge case of your choosing.
4. Run `adk eval` against it and read the actual pass/fail output — verify current `adk eval` invocation syntax against official docs if it's unclear, don't guess at the flags.
5. Fix anything the eval run reveals as broken before moving on.

## Acceptance Criteria

- At least two agents exist, composed so the root agent can delegate based on description.
- A live test shows correct routing for both an FAQ case and an escalation case.
- An `adk eval` set with at least 3 cases exists and was actually run, with real (not assumed) pass/fail output reviewed.

## Common Mistakes

- Writing sub-agent descriptions vaguely enough that routing becomes unpredictable — this is where Lesson 2's description-vs-instruction distinction pays off or doesn't.
- Treating "I manually tried the eval cases once" as equivalent to running `adk eval` — the point of this lesson is the repeatable automated check, not one-off manual testing.
- Skipping the fix step when `adk eval` reveals a failure — a required lesson isn't complete with a known-failing eval case left unaddressed.
