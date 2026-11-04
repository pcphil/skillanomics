# Product Lenses (Opportunity ideas)

Load when the focus includes Opportunity. Reason as a product manager: start from the user's job, then look for where the codebase can grow to serve it. Every Opportunity needs an **anchor**: `file:line` of the nearest extension point or existing capability, or a quote from the user's Context answers. Leave out market or competitor claims that the repo or the user cannot source.

## Jobs to Be Done

- Restate what users hire the product for, in their words from Context.
- Find steps of that job the product leaves to the user (copy-paste, spreadsheets, manual follow-up).
- Search the codebase for the existing feature closest to each unserved step. That is the anchor.

## Activation Gaps

- First-run path: what must a new user do before getting value? Search onboarding, signup, and setup code.
- Steps where users likely stall (long forms, required config, empty first screens).
- Missing quick wins: templates, sample data, guided first action.

## Retention Gaps

- What brings users back? Search for notifications, digests, history, saved items, and sharing.
- Dead ends after the main task completes (no next step, no summary, no export).
- Data users accumulate but cannot reuse, search, or revisit.

## Adjacent Features

- Capabilities one small step from what exists: import for an existing export, an API for an existing UI action, bulk for an existing single action.
- Integrations the code already half-supports (webhooks, config hooks, plugin points).

## Table-Stakes Gaps

- Features users of this kind of product commonly expect, visible from the repo's own docs, issues, TODOs, or the user's statements.
- Mark the source of every claim. Drop the ones with no source.

## Extension Points

- Interfaces, registries, config maps, and route tables where a new capability plugs in cheaply.
- Each extension point cited with `file:line` makes a strong anchor and a small first slice.

## Riff Prompts

Use these to build on the user's tangents:

- "Who else could use this if we changed X?"
- "What would the smallest version of this look like?"
- "What happens right after the user finishes this task?"
- "What does the user do outside the product to finish this job?"

## Mini-Grill for Accepted Opportunities

Ask each accepted Opportunity, with a recommended answer for every question:

1. **Problem**: what pain does this remove, in one sentence?
2. **User**: who benefits first?
3. **Success signal**: what observable change shows it worked?
4. **Smallest first slice**: what is the least we could build that still tests the idea? Point to the anchor.
