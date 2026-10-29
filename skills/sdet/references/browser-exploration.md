# Live browser exploration

Loaded when a browser-control tool is available in the agent environment (for example, Playwright MCP) and a phase needs the live app: UI page objects, API discovery, journey mapping, or e2e debugging.

The browser is for **scouting**. What you find goes into test code that the project's runner executes in CI. Clicking through a flow in the browser tool is not a test and never counts as one in Verify.

## 1. Before the first navigation

1. **Confirm target.** Ask the user to confirm the site (base URL and environment: local / staging / production) and the account to use. Navigate only to confirmed sites.
2. **Profile guard.** Exploration runs in a separate automation profile, never in the user's personal browser. Before acting, check for signs of a personal profile: logged into personal accounts (email, banking, social), unrelated open tabs, or the tool running attached to an existing browser (extension or remote-debugging mode). If any sign appears, stop and ask the user before continuing.
3. **Directions.** Ask the user which journeys or pages to explore and what to look for. Offer the P0 journeys from the strategy as a starting list. If the user says to go ahead without specifics, walk the P0 journeys.

## 2. Login

- **Default:** the user logs in by hand in the visible browser window, then says to continue. The password never enters the transcript.
- **Dedicated test accounts only:** the agent may type credentials read from environment variables (`E2E_USER`, `E2E_PASSWORD`). Use this only for accounts that exist purely for testing.
- The automation profile persists sessions between runs; check whether you are already logged in before asking.

## 3. Rules by environment

| Env | Allowed in the browser |
|-----|------------------------|
| local | full interaction: forms, data creation, settings |
| staging | full interaction; record every entity created so it can be cleaned up |
| production | read-only navigation, snapshots, reading network traffic |

**Production writes:** an action that writes (submitting a form, purchasing, deleting, changing settings) happens on production only when the user explicitly asks for that specific action. Before acting, state exactly what will happen ("This submits a real contact form on production as <account>") and wait for a yes. Each write needs its own confirmation — one approval never covers the next action.

Report every entity created on staging or production at the end of exploration so it can be removed.

## 4. Recipes

### 4.1 Selector discovery (for page objects)

1. Navigate to the screen.
2. Take an accessibility snapshot of the page.
3. For each element the page object needs, pick the locator from the snapshot in this order: role + accessible name → label → placeholder/text → test ID.
4. Note elements with no accessible name — these need a `data-testid` added to the app; tell the user.
5. Drive the journey step by step and snapshot after each transition, so each page object's locators come from the state the test will actually see.
6. Write the page object with exactly those locators.

### 4.2 Journey mapping (for Assess / Strategy)

Walk each directed journey and record:

- Steps and the URL after each step.
- Forms: fields, required markers, client-side validation messages.
- Branches and error states seen (empty results, permission denied, validation errors).
- Console errors and warnings.
- Anything slow or visibly broken — candidate risks for the strategy.

### 4.3 API discovery (for API tests and schemas)

1. Walk the journey with network inspection on.
2. For each request the app makes to its own backend, record: method, path pattern, status, request body shape, response body shape, auth mechanism (cookie, bearer header).
3. Ignore third-party requests (analytics, CDNs, ads).
4. Turn response bodies into draft schemas under `tests/api/schemas/` — field names and types only. Prefer an OpenAPI spec or typed models in the codebase when they exist; use captured shapes to fill gaps or check the spec against reality.
5. Strip tokens, cookies, emails, names, and other personal data from anything written down. Raw captures stay out of the repo.

### 4.4 Debugging a failing e2e test (in Verify)

1. Read the failure: assertion, locator, and step.
2. Reproduce the same steps in the browser against the same environment.
3. At the failing step, snapshot the page and check console messages and network responses.
4. Classify: selector changed (fix the page object), timing (replace with a condition-based wait or web-first assertion), data missing (fix the fixture), or real product bug (report as a finding with the evidence).
5. Fix the test code, then rerun it through the project's runner — the fix is verified only by the runner, not by the browser.

## 5. Without a browser tool

Use these instead, in order:

1. Read the component source and templates for roles, labels, and test IDs.
2. Suggest the user run `npx playwright codegen <url>` (or `playwright codegen <url>` for Python) and paste the generated locators.
3. Use the Playwright trace viewer on a failing run: `npx playwright show-trace <trace.zip>`.
4. For API shapes: the OpenAPI spec, route handlers in the codebase, or responses the user pastes.

Mention in the Assess summary, once: "No browser-control tool detected; with one (e.g. Playwright MCP), selector discovery and API capture can run against the live app." Then continue with the fallbacks.

## 6. Exploration note

After exploring, add or update the **Journeys & endpoints** section of `tests/TESTING.md` so later runs can skip re-exploring:

```markdown
## Journeys & endpoints

Explored: <date>, <env>, <account role>

### Journeys
| Journey | Steps (URL path) | Key elements | Notes |
|---------|------------------|--------------|-------|
| Login | /login → /dashboard | Email, Password, "Log in" button | error text "Invalid credentials" |

### Endpoints
| Method | Path | Status | Auth | Schema |
|--------|------|--------|------|--------|
| POST | /api/session | 201 | none → sets cookie | schemas/session.json |
| GET | /api/orders?page= | 200 | cookie | schemas/order-list.json |

### Gaps
- <elements without accessible names, needing data-testid>
- <entities created during exploration, for cleanup>
```
