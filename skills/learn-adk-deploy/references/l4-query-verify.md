# Lesson 4: Query & Verify

## Concept

A deployment isn't proven working until you've actually queried it and gotten a real response back from the running Agent Runtime — a deploy that "completed successfully" per the CLI/SDK output can still fail on first real use (bad IAM, wrong region, misconfigured dependency).

## Task

**Live path:**
1. Using the resource identifier from Lesson 3, send a real query to the deployed agent (the same kind of async streaming query used for local testing in Lesson 2, now pointed at the deployed resource instead of the local `AdkApp`).
2. Confirm the response is a genuine, coherent answer — not an error, not an empty response.
3. Try a second query that exercises one of the agent's tools (e.g. the FAQ lookup) to confirm tool-calling still works once deployed, not just plain conversation.

**Dry-run path (if Lesson 3 was conceptual only):**
1. Read through documentation examples of querying a deployed Agent Runtime resource and describe, in your own words, what the request/response shape looks like and how it differs from the local `AdkApp` query in Lesson 2.

## Acceptance Criteria

- Live path: at least one real query against the deployed agent returned a genuine response, and one query exercising a tool call succeeded.
- Dry-run path: the learner can describe the querying pattern and how it differs from local testing.

## Common Mistakes

- Treating "the deploy command exited successfully" as proof the agent works — always confirm with an actual query.
- Only testing plain conversation and never confirming tool calls still work through the deployed path — tool wiring is a common place for deploy-only failures.
- For the dry-run path, skipping the comparison to local querying — the point is understanding what's actually different about hitting a deployed resource, not just reading unrelated docs.
