# Lesson 6: Callbacks & Streaming

## Concept

**Callbacks** are hook points around an agent's execution — before/after the agent runs, before/after a model call, before/after a tool call. They're where you add cross-cutting behavior without cluttering the agent's core instruction: logging, guardrails (e.g. block a tool call if an argument looks wrong), or injecting extra context.

**Streaming** is about how responses are delivered: instead of waiting for the full response and returning it in one chunk, the agent can emit partial output as the model generates it — the difference between a chat UI that "types" a response live vs. one that shows nothing until the answer is complete.

For the support agent, a natural pairing: use a callback to log every tool call (useful for debugging Lesson 3's FAQ tool and later multi-agent behavior), and confirm streaming actually delivers partial output rather than one final blob.

## Task

1. Add a callback (e.g. a `before_tool_callback` or `after_tool_callback`) that logs which tool was called and with what arguments.
2. Trigger the FAQ tool via `adk run` and confirm the callback's log output appears alongside the normal conversation.
3. Confirm (via `adk web` or the streaming-capable run mode) that responses stream incrementally rather than appearing all at once — observe and describe what you see.

## Acceptance Criteria

- At least one callback is wired in and demonstrably fires (visible log output) during a real tool call.
- The learner has observed and can describe the difference between a streamed response and a non-streamed one.

## Common Mistakes

- Adding a callback that never actually fires because it's attached to the wrong hook point (e.g. a tool callback when no tool call happens in the test) — verify by triggering a path that's guaranteed to hit it.
- Confusing "the terminal prints text quickly" with true streaming — check for genuinely incremental delivery, not just fast non-streamed output.
- Using a callback to duplicate what the instruction already handles — callbacks are for cross-cutting concerns, not core behavior logic.
