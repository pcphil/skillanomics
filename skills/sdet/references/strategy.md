# Strategy

Loaded at `[SDET: Strategy]`. Turns the Assess summary into an agreed plan: what is at risk, which layer proves it, which tools, and the concrete cases. Every later phase traces back to this.

## 1. System-under-test and boundaries

State in two or three lines:

- What is under test (service, feature, endpoint set, user flow).
- What is in and out of scope for this run.
- External dependencies and how tests will treat each one: real (in a container), stubbed at the boundary, or out of scope.

## 2. Risk areas

List where the system is most likely to break or most costly when it does. Rate each **High / Medium / Low** on impact × likelihood. Sources of risk to check:

- **Money and data** — payments, balances, writes that cannot be undone, data loss.
- **Auth and access** — login, sessions, roles, tenant isolation, leaked existence of records.
- **Input handling** — validation, encoding, size limits, malformed payloads.
- **State transitions** — order lifecycles, retries, idempotency, concurrent updates.
- **Integrations** — third-party APIs, queues, webhooks, timeouts.
- **Load-sensitive paths** — hot endpoints, search, checkout, anything with N+1 risk.
- **Recent change** — areas with recent commits or open bugs (check git history).

High risks get coverage at more than one layer. Low risks may get none — say so explicitly.

## 3. Layer split

Choose layers per risk, with a one-line rationale each. Default shape is a pyramid:

| Layer | Share | Proves |
|-------|-------|--------|
| Unit | most | pure logic, validators, calculations, edge permutations |
| Integration / API | middle | contracts at the boundary: status codes, error shapes, schemas, DB effects |
| UI / e2e | fewest | a handful of critical user journeys end to end |
| Performance | targeted | latency and error rate under expected and peak load on hot paths |

**Push-down rule:** if a lower layer can prove it, test it there. Input permutations never go through the browser.

## 4. Tools per stack

| Concern | Python | JS / TS |
|---------|--------|---------|
| Runner | pytest | Vitest (unit/integration), Playwright Test (e2e/API) |
| HTTP API | httpx + pytest | Playwright `request` fixture or Vitest + `fetch` |
| Schema | jsonschema / pydantic | ajv / zod |
| UI / e2e | pytest-playwright | @playwright/test |
| Data | Faker, factory_boy | @faker-js/faker, factory functions |
| Containers | testcontainers-python | testcontainers-node |
| Performance | Locust | Locust (black-box HTTP, stack-independent) |
| Reports | JUnit XML + pytest-html | JUnit XML + Playwright HTML |

When the repo already uses a different tool for a concern (e.g. Jest instead of Vitest), keep the repo's tool.

## 5. Environments

Record per environment: name, base URL env var, what may run there.

| Env | Allowed |
|-----|---------|
| local | everything |
| staging | everything, including load within agreed limits |
| production | read-only smoke tests only, after explicit user confirmation; no load |

## 6. Case table

One row per test. IDs prefix by layer: `U-` unit, `I-` integration/API, `E-` e2e, `P-` performance.

| ID | Title | Layer | Risk | Priority | Preconditions | Steps / Inputs | Expected result |
|----|-------|-------|------|----------|---------------|----------------|-----------------|

- **Priority:** P0 = blocks release, gates every merge; P1 = full regression; P2 = nice to have.
- Every requirement gets a happy path plus at least one negative or edge case.
- Every row states its expected result concretely (status code, field value, visible text, latency threshold).
- Check the finished table back against section 2: each High risk has at least one P0 row.

## 7. Output

**Full pipeline / no framework:** sections 1–6 in full, then stop for user sign-off.

**Targeted:** a short plan — risks covered, layer(s), and the case rows for this task — then continue.

## TESTING.md template

Written to `tests/TESTING.md` in the target repo after full-pipeline sign-off. Keep it short; it is the contract later runs follow.

```markdown
# Testing

## Stack
- Runner(s): <pytest | Vitest | Playwright Test>
- Layers in use: <unit, api, e2e, perf>

## Layout
<tree of test directories and what lives where>

## Conventions
- Test names carry case IDs: <example>
- Markers / tags: smoke (P0, gates merge), regression (full), api, e2e, slow
- Data: each test creates and cleans up its own data via <factories/fixtures path>
- Selectors: role/label first, test IDs as fallback

## Environments
| Env | Base URL var | Allowed |
|-----|--------------|---------|
| local | BASE_URL | all |
| staging | BASE_URL | all, load within limits |
| production | BASE_URL | read-only smoke, explicit confirmation |

## Running
<exact commands per layer>

## CI
<workflow file, what gates merge, what runs nightly>

## Risk map
<High risks → case IDs covering them>

## Journeys & endpoints
<filled after live browser exploration — format in references/browser-exploration.md>
```
