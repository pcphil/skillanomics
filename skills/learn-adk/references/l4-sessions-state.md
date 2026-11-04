# Lesson 4: Sessions & State

## Concept

So far each `adk run` message has been handled fresh — the agent doesn't remember what was said two turns ago beyond what's in the immediate conversation. ADK's **Session** is the object that groups a sequence of turns together, and **State** is a key-value dict attached to a session that the agent (and its tools) can read and write across turns within that session.

This matters concretely for a support agent: if a user says "my order number is 12345" in turn 1 and "what's its status?" in turn 2, the agent needs turn 2 to still know the order number — that's state, not memory (memory, Lesson 5, is about persisting things *across* sessions, not just within one).

`adk run`/`adk web` manage a session for you automatically per conversation; the concept to learn here is what's actually happening underneath and how to read/write state deliberately from a tool.

## Task

1. Add a stateful tool that reads and writes session state — e.g. a `remember_order_number` tool that stores an order number the user mentions, and have `lookup_faq` (or a new tool) read it back when relevant.
2. Update the instruction so the agent proactively asks for and stores the order number once, instead of asking again every turn.
3. Test a multi-turn conversation via `adk run`: mention an order number in turn 1, then ask a follow-up in turn 2 that requires the agent to recall it without you repeating it.
4. Confirm from the conversation that state persisted correctly across those turns.

## Acceptance Criteria

- At least one tool reads or writes session state (not just local variables or the FAQ dict).
- A multi-turn test shows information given in an earlier turn being used correctly in a later turn within the same session.
- The learner can state, in their own words, the difference between session state and memory (Lesson 5).

## Common Mistakes

- Storing "state" in a Python global/module-level variable instead of actual session state — this breaks across separate sessions/users and defeats the purpose of the lesson.
- Confusing "the model has more conversation history in context" with "state was deliberately read/written" — the acceptance criteria requires an actual state read/write, not just a longer chat transcript.
- Forgetting to test with a genuinely separate later turn — testing everything in one message doesn't exercise state persistence at all.
