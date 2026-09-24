# Lesson 5: Memory

## Concept

Where Session/State (Lesson 4) lives and dies with a single conversation, **Memory** is for information that should persist and be retrievable *across* sessions — e.g. a returning support user whose past issue should inform a new conversation days later.

The core idea to teach: memory is a service the agent (or a memory-lookup tool) queries, separate from the live conversation. A session's contents can be committed to memory at the end of a conversation, then searched/recalled in a future, unrelated session.

Keep this lesson conceptually scoped — the goal is understanding *when* to reach for memory vs. state, and wiring up a basic memory read/write path for the support agent, not building a production-grade memory architecture.

## Task

1. Add memory to the support agent: after a conversation ends (or at a natural checkpoint), save a short summary of the interaction (e.g. "user asked about refund policy on 2026-09-14") to memory.
2. Add a way for the agent to check memory at the start of a new session — e.g. a tool that looks up prior interactions for context before responding.
3. Test it: run one `adk run` session, end it, start a fresh `adk run` session, and confirm the agent can reference something from the prior session via memory (not via session state, which wouldn't carry over).
4. Explain in one sentence why this had to be memory and not state.

## Acceptance Criteria

- The agent writes something to memory during or after a session.
- The agent reads from memory in a *separate, later* session and demonstrably uses it.
- The learner can articulate why state alone wouldn't have worked for this scenario.

## Common Mistakes

- Testing memory recall within the same `adk run` session — that would also work with plain state and doesn't prove memory is doing anything.
- Writing memory content that's too generic to be useful on recall ("user talked to the agent") instead of something specific enough to matter later.
- Treating memory as a place to dump everything — for this lesson, one meaningful piece of persisted context is enough to prove the mechanism works.
