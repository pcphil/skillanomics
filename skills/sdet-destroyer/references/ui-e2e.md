# UI / e2e tests (Playwright)

Loaded at `[SDET: UI]`. e2e tests drive a real browser against a running app to prove the critical user journeys. They are the slowest and most fragile layer — keep them few and make each one count.

Stack: Python uses `pytest-playwright` (sync API, `test_*.py`). TS uses `@playwright/test` (`*.spec.ts`). Both provide page objects through fixtures and set up data through the API clients in `tests/support/clients/` (`references/api.md`).

## 1. What belongs here

- The P0/P1 journeys from the strategy: sign-up, login, core purchase/creation flow, critical settings.
- Behavior only visible in the browser: client-side validation, navigation, rendering of server data.
- Everything else — input permutations, error codes, business rules — is proven at the unit or API layer.

## 2. Page objects

Rules are in `references/architecture.md` §7. Shapes:

### Python

```python
# tests/e2e/pages/login_page.py
from playwright.sync_api import Page
from tests.e2e.pages.dashboard_page import DashboardPage

class LoginPage:
    def __init__(self, page: Page):
        self.page = page
        self.email = page.get_by_label("Email")
        self.password = page.get_by_label("Password")
        self.submit = page.get_by_role("button", name="Log in")

    def open(self) -> "LoginPage":
        self.page.goto("/login")
        return self

    def login(self, email: str, password: str) -> DashboardPage:
        self.email.fill(email)
        self.password.fill(password)
        self.submit.click()
        return DashboardPage(self.page)
```

```python
# tests/e2e/conftest.py (page object fixtures)
import pytest
from tests.e2e.pages.home_page import HomePage
from tests.e2e.pages.login_page import LoginPage
from tests.e2e.pages.orders_page import OrdersPage

@pytest.fixture
def home_page(page):
    return HomePage(page)

@pytest.fixture
def login_page(page):
    return LoginPage(page)

@pytest.fixture
def orders_page(page):
    return OrdersPage(page)
```

### TS

```ts
// tests/e2e/pages/login-page.ts
import type { Locator, Page } from "@playwright/test";
import { DashboardPage } from "./dashboard-page";

export class LoginPage {
  readonly email: Locator;
  readonly password: Locator;
  readonly submit: Locator;

  constructor(readonly page: Page) {
    this.email = page.getByLabel("Email");
    this.password = page.getByLabel("Password");
    this.submit = page.getByRole("button", { name: "Log in" });
  }

  async open() {
    await this.page.goto("/login");
    return this;
  }

  async login(email: string, password: string) {
    await this.email.fill(email);
    await this.password.fill(password);
    await this.submit.click();
    return new DashboardPage(this.page);
  }
}
```

```ts
// tests/e2e/fixtures.ts — extends the API fixtures so UI tests get clients for setup
import { test as apiTest } from "../api/fixtures";
import { HomePage } from "./pages/home-page";
import { LoginPage } from "./pages/login-page";
import { OrdersPage } from "./pages/orders-page";

export const test = apiTest.extend<{ homePage: HomePage; loginPage: LoginPage; ordersPage: OrdersPage }>({
  homePage: async ({ page }, use) => use(new HomePage(page)),
  loginPage: async ({ page }, use) => use(new LoginPage(page)),
  ordersPage: async ({ page }, use) => use(new OrdersPage(page)),
});
export { expect } from "@playwright/test";
```

## 3. Selectors

In order of preference:

1. `getByRole` / `get_by_role` with accessible name.
2. `getByLabel`, `getByPlaceholder`, `getByText`.
3. `getByTestId` / `get_by_test_id` when no accessible name exists — add a `data-testid` to the app and tell the user.

Selectors are drafted from the app's real markup — read the component source or snapshot the page before writing them. When a browser-control tool is available, take selectors from a live accessibility snapshot (`references/browser-exploration.md`, selector discovery).

## 4. Synchronization

Use web-first assertions that auto-retry (`toBeVisible`, `toHaveURL`, `toHaveText` / `to_be_visible`, `to_have_url`). Wait on a condition: `waitForURL`, `waitForResponse`. When tempted by a sleep, write the assertion that describes the state you are waiting for.

## 5. Auth: storage state per role

Log in through the UI in exactly one test (E-01 proves the login journey). Every other test starts already authenticated as its role, with storage state produced **through the API** once per session. The login path and payload are placeholders — use what Assess found. If the app keeps its session in localStorage rather than cookies, write the token into the state's `origins` entry instead.

### Python

httpx serves the API clients; Playwright's request context is used here only because it writes browser storage state directly.

```python
# tests/support/fixtures/auth.py — registered via pytest_plugins
import pytest

@pytest.fixture(scope="session")
def storage_state_for(playwright, tmp_path_factory, env_settings):
    states: dict[str, str] = {}

    def get(role: str) -> str:
        if role not in states:
            creds = env_settings.role(role)
            ctx = playwright.request.new_context(base_url=env_settings.api_base)
            resp = ctx.post("/auth/login", data={"username": creds.username, "password": creds.password})
            assert resp.ok, f"login failed for role {role}: {resp.status}"
            path = tmp_path_factory.mktemp("auth") / f"{role}.json"
            ctx.storage_state(path=path)
            ctx.dispose()
            states[role] = str(path)
        return states[role]

    return get
```

```python
# tests/e2e/conftest.py (overrides of pytest-playwright fixtures live here, not in plugin modules)
import pytest

@pytest.fixture(scope="session")
def base_url(env_settings):
    return env_settings.require("base_url")

@pytest.fixture
def browser_context_args(browser_context_args, storage_state_for, request):
    marker = request.node.get_closest_marker("role")
    role = marker.args[0] if marker else "member"
    if role is None:
        return browser_context_args  # anonymous
    return {**browser_context_args, "storage_state": storage_state_for(role)}
```

Choose the role per test with `@pytest.mark.role("admin")`; `@pytest.mark.role(None)` runs anonymously; no marker means `member`. Under xdist each worker produces its own storage state — sessions stay isolated per worker.

### TS

```ts
// tests/e2e/auth.setup.ts — one storage state file per role, via the API
import { test as setup } from "@playwright/test";
import { roleNames, roles, settings } from "../settings";

for (const role of roleNames) {
  setup(`authenticate as ${role}`, async ({ request }) => {
    const res = await request.post(`${settings.apiBase}/auth/login`, { data: roles[role] });
    if (!res.ok()) throw new Error(`login failed for role ${role}: ${res.status()}`);
    await request.storageState({ path: `.auth/${role}.json` });
  });
}
```

```ts
// default role for all e2e projects — playwright.config.ts, inside each browser project's `use`
storageState: ".auth/member.json",

// per file or describe block
test.use({ storageState: ".auth/admin.json" });          // as admin
test.use({ storageState: { cookies: [], origins: [] } }); // anonymous
```

`.auth/` holds live session cookies — keep it git-ignored.

## 6. Data setup through the API

**Set up state via API, assert via UI.** Create what a journey needs with the `*_ok` / `*Ok` client methods, then drive and assert in the browser. Clean up after.

```python
# tests/e2e/test_orders.py
import pytest
from playwright.sync_api import expect
from tests.factories.order import order_payload

@pytest.fixture
def order(member_api):
    created = member_api.orders.create_ok(order_payload())
    yield created
    member_api.orders.delete(created.id)

# E-04 member sees their order in the order list
@pytest.mark.e2e
@pytest.mark.writes
def test_e04_order_listed(orders_page, order):
    orders_page.open()
    expect(orders_page.row(order.id)).to_be_visible()
```

```ts
// tests/e2e/orders.spec.ts
import { test, expect } from "./fixtures";
import { orderPayload } from "../factories/order";

test("E-04 member sees their order in the order list @writes", async ({ memberApi, ordersPage }) => {
  const order = await memberApi.orders.createOk(orderPayload());
  try {
    await ordersPage.open();
    await expect(ordersPage.row(order.id)).toBeVisible();
  } finally {
    await memberApi.orders.delete(order.id);
  }
});
```

The API role used for setup must match the browser role (`member_api` with the default `member` storage state) so the UI shows the data the test created.

Never depend on data someone created by hand in staging.

## 7. Browsers

- Merge gate: Chromium.
- Nightly: Firefox, WebKit, and a mobile viewport (TS projects `firefox`, `webkit`, `mobile`; Python `--browser firefox --browser webkit` and a `--device "Pixel 7"` run).
- The strategy can promote a browser onto the gate when its audience share or past bugs make it high-risk.

## 8. Specialized checks (when the strategy calls for them)

- **Accessibility:** `@axe-core/playwright` / `axe-playwright-python` on key screens; fail on new serious/critical violations.
- **Visual regression:** `toHaveScreenshot()` with committed baselines and masked dynamic regions. Gate selectively — rendering differs across OS.

## 9. Discipline

- One row → one test; e2e only for the journeys the strategy assigned.
- Each test independent: own data, own state, any order.
- Each test ends on an explicit assertion of the expected result.
