# Lesson 5: Governance Overview (Overview Only)

This lesson is intentionally an overview, not a deep dive. Do not expand into org policy design, multi-team rollout planning, or fine-grained IAM architecture — those are out of scope for this skill.

## Concept

Once an agent exists as an Agent Runtime resource, Gemini Enterprise Agent Platform (and the underlying Agentspace layer) adds an enterprise management surface on top of raw deployment:

- **Registration** — making the deployed agent discoverable/usable within an enterprise Agentspace app, rather than only reachable via direct API calls.
- **Access control** — who inside the organization can invoke or manage the agent, layered on top of the base IAM roles used for deployment itself.
- **Monitoring** — visibility into usage, errors, and performance of the deployed agent over time, distinct from the one-off query checks done in Lesson 4.

The goal here is recognizing these exist and roughly what each is for, so the learner knows what to look into if their organization needs it — not becoming proficient in configuring them.

## Task

1. Read through current official documentation on registering an ADK agent with Gemini Enterprise/Agentspace, and summarize in a few sentences what registration actually adds beyond having a working Agent Runtime resource.
2. Identify, at a glance, where access control settings for a registered agent would live (without necessarily configuring them).
3. Identify, at a glance, where monitoring/usage data for the deployment would be visible.

## Acceptance Criteria

- The learner can state, in their own words, what registration adds beyond a bare deployed Agent Runtime resource.
- The learner can point to where access control and monitoring surfaces exist, even without having configured them in depth.

## Common Mistakes

- Trying to actually configure fine-grained access control or org-wide policy here — that's explicitly out of scope; note it as a "beyond this course" item and move on.
- Confusing this enterprise-level access control with the base IAM roles from Lesson 1 — they're related but layered, not the same thing.
