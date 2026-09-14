# Capstone: Deployment Mechanics End-to-End

## Goal

Confirm the full deployment path was walked through coherently, either as a real live deployment or a complete conceptual dry-run — not a new build, the accumulated result of Lessons 1-5.

## Acceptance Criteria

The capstone is complete when all of the following are true:

1. **GCP bootstrap**: a specific GCP project with billing, both required APIs, and both required IAM roles was confirmed (Lesson 1).
2. **AdkApp wrapping**: the agent was wrapped in `AdkApp` and tested locally before any deploy action (Lesson 2).
3. **Agent Engine deploy**: either (a) a live deployment completed with a captured resource identifier and the cost checkpoint was explicitly handled, or (b) the dry-run path was completed with the learner able to explain each deploy argument and what success looks like (Lesson 3).
4. **Query & verify**: either (a) the deployed agent was queried live, including a query that exercised a tool call, or (b) the dry-run path's querying pattern was explained and compared to local testing (Lesson 4).
5. **Governance overview**: the learner can state what registration adds and where access control/monitoring surfaces live, without having gone deep into configuring them (Lesson 5).

State clearly which path was taken (live vs. dry-run) — both are valid completions of this skill; live deployment is the fuller experience but was never required.

## Completion

Summarize what was done: which agent was deployed (or scaffolded), live or dry-run outcome, and the governance concepts covered. Ask if the learner wants to explore governance features hands-on, extend the deployment further, or return to `learn-adk` to keep building the underlying agent.
