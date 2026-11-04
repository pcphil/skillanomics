# Common Lenses (always load)

Quality-of-life gaps that apply to any project type. Search the codebase for each; cite `file:line` for every hit.

## Dev Setup

- README missing a one-command setup, or steps that assume undocumented tools.
- No example env file, or env vars used in code but absent from docs.
- No seed or fixture data for local runs.
- No single command to run tests, lint, or start the project.

## Config

- Magic values inline that users would want to change (timeouts, page sizes, limits).
- Config errors that fail late or silently instead of at startup.
- Defaults that force every user to override the same setting.

## Logging and Diagnostics

- Errors swallowed or logged without context (no ids, inputs, or cause).
- Inconsistent log formats or levels.
- No way to turn on verbose output when debugging.

## Scripts and Tooling

- Repeated manual steps a script could cover (reset db, generate types, release).
- Task runners or scripts undocumented or scattered.

## Consistency

- Same concept named differently across files.
- Similar operations behaving differently (one validates, its sibling does not).
