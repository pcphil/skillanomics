# UI / e2e tests (Playwright)

Loaded at `[SDET: UI]`. e2e tests drive a real browser against a running app to prove the critical user journeys. They are the slowest and most fragile layer — keep them few and make each one count.

Pick the binding that matches the repo: `@playwright/test` (TS, `*.spec.ts`) or `pytest-playwright` (Python, `test_*.py`, runs under pytest).

## 1. What belongs here

- The P0/P1 journeys from the strategy: sign-up, login, core purchase/creation flow, critical settings.
- Behavior only visible in the browser: client-side validation, navigation, rendering of server data.
- Everything else — input permutations, error codes, business rules — is proven at the unit or API layer.

## 2. Page objects

### TS

```ts
// tests/e2e/pages/login-page.ts
import { type Page, type Locator } from "@playwright/test";
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
// tests/e2e/fixtures.ts
import { test as base } from "@playwright/test";
import { LoginPage } from "./pages/login-page";

export const test = base.extend<{ loginPage: LoginPage }>({
  loginPage: async ({ page }, use) => use(new LoginPage(page)),
});
export { expect } from "@playwright/test";
```

```ts
// tests/e2e/login.spec.ts
import { test, expect } from "./fixtures";

test("E-01 valid login lands on dashboard @smoke", async ({ loginPage }) => {
  const dashboard = await (await loginPage.open()).login(process.env.E2E_USER!, process.env.E2E_PASSWORD!);
  await expect(dashboard.heading).toBeVisible();
});
```

### Python

```python
# tests/e2e/pages/login_page.py
from tests.e2e.pages.dashboard_page import DashboardPage

class LoginPage:
    def __init__(self, page):
        self.page = page
        self.email = page.get_by_label("Email")
        self.password = page.get_by_label("Password")
        self.submit = page.get_by_role("button", name="Log in")

    def open(self):
        self.page.goto("/login")
        return self

    def login(self, email, password):
        self.email.fill(email)
        self.password.fill(password)
        self.submit.click()
        return DashboardPage(self.page)
```

```python
# tests/e2e/test_login.py
import os
import pytest
from playwright.sync_api import expect
from tests.e2e.pages.login_page import LoginPage

# E-01 valid login lands on dashboard
@pytest.mark.e2e
@pytest.mark.smoke
def test_e01_valid_login(page):
    dashboard = LoginPage(page).open().login(os.environ["E2E_USER"], os.environ["E2E_PASSWORD"])
    expect(dashboard.heading).to_be_visible()
```

Base URL for Python: `--base-url` in `pytest.ini` `addopts`, or set from `BASE_URL` in CI.

## 3. Selectors

In order of preference:

1. `getByRole` / `get_by_role` with accessible name.
2. `getByLabel`, `getByPlaceholder`, `getByText`.
3. `getByTestId` / `get_by_test_id` when no accessible name exists — add a `data-testid` to the app and tell the user.

Selectors are drafted from the app's real markup — read the component source or snapshot the page before writing them.

## 4. Synchronization

Use web-first assertions that auto-retry (`toBeVisible`, `toHaveURL`, `toHaveText` / `to_be_visible`, `to_have_url`). Wait on a condition: `waitForURL`, `waitForResponse`. When tempted by a sleep, write the assertion that describes the state you are waiting for.

## 5. Auth reuse

Log in through the UI once (E-01 proves it); every other test starts authenticated.

```ts
// tests/e2e/auth.setup.ts — run as a setup project; other projects depend on it
import { test as setup } from "@playwright/test";

setup("authenticate", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill(process.env.E2E_USER!);
  await page.getByLabel("Password").fill(process.env.E2E_PASSWORD!);
  await page.getByRole("button", { name: "Log in" }).click();
  await page.waitForURL("**/dashboard");
  await page.context().storageState({ path: ".auth/user.json" });
});
```

```ts
// playwright.config.ts projects (excerpt)
projects: [
  { name: "setup", testMatch: /auth\.setup\.ts/ },
  { name: "chromium", use: { ...devices["Desktop Chrome"], storageState: ".auth/user.json" }, dependencies: ["setup"] },
],
```

```python
# tests/e2e/conftest.py
import os
import pytest

@pytest.fixture(scope="session")
def browser_context_args(browser_context_args, tmp_path_factory, browser, base_url):
    state = tmp_path_factory.mktemp("auth") / "state.json"
    ctx = browser.new_context(base_url=base_url)
    page = ctx.new_page()
    page.goto("/login")
    page.get_by_label("Email").fill(os.environ["E2E_USER"])
    page.get_by_label("Password").fill(os.environ["E2E_PASSWORD"])
    page.get_by_role("button", name="Log in").click()
    page.wait_for_url("**/dashboard")
    ctx.storage_state(path=state)
    ctx.close()
    return {**browser_context_args, "storage_state": str(state)}
```

Faster still: obtain a token via the API and write storage state directly. Rule: **set up state via API, assert via UI.**

Add `.auth/` to `.gitignore` — storage state holds live session cookies.

## 6. Test data in e2e

Create the data a journey needs through the API in a fixture before the test, and delete it after. Never depend on data someone created by hand in staging.

## 7. Specialized checks (when the strategy calls for them)

- **Accessibility:** `@axe-core/playwright` / `axe-playwright-python` on key screens; fail on new serious/critical violations.
- **Visual regression:** `toHaveScreenshot()` with committed baselines and masked dynamic regions. Gate selectively — rendering differs across OS.

## 8. Discipline

- One row → one test; e2e only for the journeys the strategy assigned.
- Each test independent: own data, own state, any order.
- Each test ends on an explicit assertion of the expected result.
