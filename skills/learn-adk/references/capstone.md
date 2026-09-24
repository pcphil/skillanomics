# Capstone: Working Support Agent

## Goal

A single, working support agent that demonstrates every mechanic taught in Lessons 1-7 integrated together — not a new build, the accumulated project from those lessons brought to a coherent finished state.

## Acceptance Criteria

The capstone is complete only when all of the following are true and demonstrated live via `adk run` (or `adk web`):

1. **Agent basics**: a clearly named agent with a real, specific `description` and `instruction` (Lesson 2).
2. **Tools**: the FAQ knowledge-lookup tool works and is demonstrably invoked for FAQ-relevant questions (Lesson 3).
3. **Sessions/State**: a multi-turn conversation correctly carries information from an earlier turn to a later one within the same session (Lesson 4).
4. **Memory**: something learned in one session is correctly recalled in a separate, later session (Lesson 5).
5. **Callbacks/streaming**: at least one callback fires and is observable, and streaming behavior has been confirmed (Lesson 6).
6. **Multi-agent + eval (required)**: the root agent correctly delegates between at least two agents, and an `adk eval` set with at least 3 cases has been run with real pass/fail results reviewed (Lesson 7).

Walk through a single realistic support conversation end-to-end that exercises as many of these as naturally fit (e.g. multi-turn, references an earlier order number, triggers the FAQ tool, and — for one branch — triggers escalation to the second agent).

## Wrap-Up: Extension Note (Not a Lesson)

The FAQ knowledge tool in this capstone is intentionally simple (function-based lookup over a small in-repo set). A production support agent with a large, changing knowledge base would typically replace this with retrieval-augmented generation (RAG): embedding documents, storing them in a vector index, and retrieving the most relevant chunks at query time instead of exact keyword matching. That's a substantial topic on its own — worth knowing it exists as the natural next step, not something to build here.

## Completion

Once all criteria are demonstrated, summarize what was built (agents, tools, session/memory behavior, eval results) and offer to move on to `learn-adk-deploy` to deploy this agent to Gemini Enterprise Agent Platform.
