# Lesson 3: Agent Engine Deploy

## Concept

This is the step that actually creates and runs infrastructure — and the one that costs real money. Deployment packages your wrapped `AdkApp`, its dependencies, and stages them through a GCS bucket, then creates an Agent Engine resource that Gemini Enterprise Agent Platform exposes as an **Agent Runtime**.

**Cost checkpoint — mandatory before proceeding**: state plainly that this step (and ongoing use of the deployed agent) incurs GCP billing, with no free/sandbox tier. Confirm the learner wants to proceed before running anything. If they don't, use the dry-run path instead (see below) and still complete the lesson conceptually.

**Command surface is a verify-at-lesson-time item** — the exact deploy call and required arguments have been observed to be inconsistent across sources. Have the learner check current official docs for the precise invocation rather than trusting a hardcoded example as gospel; the shape below is illustrative of the pattern, not a guaranteed-current API surface.

## Task

**Live deploy path (only after the cost checkpoint is explicitly confirmed):**
1. Verify the current deploy invocation against official docs (e.g. `client.agent_engines.create(...)` with the app, dependency list, and a staging GCS bucket — confirm exact required arguments now, don't assume).
2. Run the deploy and wait for it to complete.
3. Note the resulting resource name/ID — needed in Lesson 4 to query it.

**Dry-run / conceptual path (if declining live deploy):**
1. Walk through what each deploy argument does (dependencies list, staging bucket, agent identity type) without executing it.
2. Read through what a successful deploy's output/resource identifier looks like from documentation, so the shape is understood even without a live run.

## Acceptance Criteria

- The cost checkpoint was explicitly stated and the learner's choice (live vs. dry-run) was made with that information, not skipped past.
- Live path: a deploy actually completed and a resource identifier was captured.
- Dry-run path: the learner can explain what each part of the deploy call does and what a successful result looks like.

## Common Mistakes

- Running the deploy command before confirming the learner actually wants to incur the cost — never silent-trigger a billable action.
- Copying an example deploy invocation verbatim without checking it against current docs — argument names/required fields have been observed to shift.
- Losing track of the resulting resource identifier — Lesson 4 needs it to query the deployment.
