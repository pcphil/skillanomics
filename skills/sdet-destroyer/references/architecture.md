# Framework architecture

Loaded at `[SDET: Scaffold]`, when the repo has no test framework. These are the structural choices that decide whether a suite scales or rots. Size them to the agreed strategy — scaffold only the layers it calls for.

List every dependency to add and ask the user before installing.

## 1. Where the suite lives

| Situation | Location | Stack |
|-----------|----------|-------|
| One service, tests ship with the code (default) | `tests/` inside the app repo | the app's stack |
| Several services (multiple app services in compose, a monorepo with several apps), or the user asks | a standalone test repo, same tree at its root | Python by default; confirm with the user |

Unit tests stay in the app repo either way. Only API and UI move to a standalone repo. Record the decision in the strategy and in `tests/TESTING.md`.

## 2. Layout

Same tree for both stacks; file extensions differ (`.py` / `.ts`).

```
tests/                          # standalone repo: this tree sits at the repo root
├── TESTING.md
├── settings.py | settings.ts   # typed settings: TARGET_ENV, base URLs, role registry
├── config/
│   ├── local.env               # non-secret values, committed
│   ├── staging.env
│   └── *.secrets.env           # optional local secrets, git-ignored
├── support/
│   ├── clients/                # one API client per resource (section 5)
│   │   └── <service>/          # standalone multi-service: group clients by service
│   └── models/                 # generated from OpenAPI, or hand-written schemas
├── factories/                  # data builders (references/test-data.md)
├── unit/                       # in-app only; or co-located src/**/*.test.ts
├── api/
│   ├── conftest.py | fixtures.ts
│   └── <service>/              # standalone multi-service: tests grouped by service
├── e2e/
│   ├── conftest.py | fixtures.ts
│   ├── auth.setup.ts           # TS only: per-role storage state
│   └── pages/
└── perf/
    └── locustfile.py           # Locust is Python on every stack
```

Root config: `pytest.ini` (Python) or `playwright.config.ts` + `vitest.config.ts` (TS). `support/` holds what both API tests and UI setup use — clients and models are shared, never duplicated.

Add to `.gitignore`: `reports/`, `.auth/`, `tests/config/*.secrets.env`, `.env`.

## 3. Typed settings

One module loads config, validates it, and fails fast naming exactly what is missing. Load order: exported environment variables win, then `config/<TARGET_ENV>.secrets.env`, then `config/<TARGET_ENV>.env`. CI exports secrets; local dev uses the git-ignored secrets file.

Roles come from the strategy (at least one). Each role reads `ROLE_<NAME>_USERNAME` / `ROLE_<NAME>_PASSWORD`.

### Python (pydantic-settings + python-dotenv)

```python
# tests/settings.py
import os
from pathlib import Path
from typing import Literal
from dotenv import load_dotenv
from pydantic import BaseModel, model_validator
from pydantic_settings import BaseSettings

CONFIG = Path(__file__).parent / "config"
TARGET_ENV = os.environ.get("TARGET_ENV", "local")
load_dotenv(CONFIG / f"{TARGET_ENV}.secrets.env", override=False)
load_dotenv(CONFIG / f"{TARGET_ENV}.env", override=False)

class Role(BaseModel):
    username: str
    password: str

class Settings(BaseSettings):
    target_env: Literal["local", "staging", "production"] = "local"
    base_url: str
    api_url: str = ""
    roles: str = "member"  # comma-separated role names

    @property
    def api_base(self) -> str:
        return self.api_url or self.base_url

    @property
    def role_names(self) -> list[str]:
        return [r.strip() for r in self.roles.split(",") if r.strip()]

    def role(self, name: str) -> Role:
        key = name.upper()
        user, pwd = os.environ.get(f"ROLE_{key}_USERNAME"), os.environ.get(f"ROLE_{key}_PASSWORD")
        if not user or not pwd:
            raise RuntimeError(f"ROLE_{key}_USERNAME / ROLE_{key}_PASSWORD missing for {self.target_env}")
        return Role(username=user, password=pwd)

    @model_validator(mode="after")
    def _all_roles_present(self):
        for name in self.role_names:
            self.role(name)
        return self

settings = Settings()
```

### TS (zod + dotenv)

```ts
// tests/settings.ts
import path from "node:path";
import dotenv from "dotenv";
import { z } from "zod";

const targetEnv = process.env.TARGET_ENV ?? "local";
const configDir = path.resolve("tests/config");
dotenv.config({ path: path.join(configDir, `${targetEnv}.secrets.env`) }); // never overrides exported vars
dotenv.config({ path: path.join(configDir, `${targetEnv}.env`) });

const env = z
  .object({
    TARGET_ENV: z.enum(["local", "staging", "production"]),
    BASE_URL: z.string().url(),
    API_URL: z.string().url().optional(),
    ROLES: z.string().default("member"),
  })
  .parse({ ...process.env, TARGET_ENV: targetEnv });

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} missing for ${targetEnv}`);
  return value;
}

export const roleNames = env.ROLES.split(",").map((r) => r.trim()).filter(Boolean);
export const roles = Object.fromEntries(
  roleNames.map((name) => {
    const key = name.toUpperCase();
    return [name, { username: required(`ROLE_${key}_USERNAME`), password: required(`ROLE_${key}_PASSWORD`) }];
  }),
) as Record<string, { username: string; password: string }>;

export const settings = {
  targetEnv: env.TARGET_ENV,
  baseUrl: env.BASE_URL,
  apiBase: env.API_URL ?? env.BASE_URL,
};
```

### Production guard

Tests that write data never run against production; only read-only smoke runs there.

```python
# tests/conftest.py
import pytest
from tests.settings import settings

def pytest_collection_modifyitems(config, items):
    if settings.target_env != "production":
        return
    skip = pytest.mark.skip(reason="production run: read-only smoke only")
    for item in items:
        if "smoke" not in item.keywords or "writes" in item.keywords:
            item.add_marker(skip)
```

```ts
// playwright.config.ts (excerpt)
grep: settings.targetEnv === "production" ? /@smoke/ : undefined,
grepInvert: settings.targetEnv === "production" ? /@writes/ : undefined,
```

## 4. Runners

| Stack | Unit | API | UI |
|-------|------|-----|----|
| Python | pytest | pytest + httpx | pytest + pytest-playwright (sync API) |
| TS | Vitest | Playwright Test (`request`) | Playwright Test |

When the repo already uses another runner for a layer, keep it.

### pytest

```ini
[pytest]
pythonpath = .
testpaths = tests
addopts = --strict-markers --junitxml=reports/junit.xml
markers =
    smoke: P0 critical path, gates every merge
    regression: full coverage, nightly / pre-release
    api: API layer
    e2e: browser layer
    writes: creates or mutates data; never runs on production
    slow: long-running, opt-in
    role(name): run the e2e test as this role; role(None) = anonymous
```

`--strict-markers` turns a mistyped marker into an error instead of a silent no-op. pytest-playwright runs Chromium by default; nightly adds `--browser firefox --browser webkit` and `--device "Pixel 7"`. Base URL for e2e: pass `--base-url "$BASE_URL"` on the command line or set `base_url` in a fixture from settings.

### Vitest (unit only)

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["tests/unit/**/*.test.ts", "src/**/*.test.ts"],
    reporters: ["default", "junit"],
    outputFile: { junit: "reports/junit-unit.xml" },
  },
});
```

### Playwright Test (API + UI)

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";
import { settings } from "./tests/settings";

export default defineConfig({
  testDir: "./tests",
  fullyParallel: true,
  retries: process.env.CI ? 1 : 0,
  reporter: [
    ["list"],
    ["junit", { outputFile: "reports/junit-pw.xml" }],
    ["html", { open: "never", outputFolder: "reports/html" }],
  ],
  use: {
    baseURL: settings.baseUrl,
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  projects: [
    { name: "api", testMatch: /api\/.*\.spec\.ts/, use: { baseURL: settings.apiBase } },
    { name: "setup", testMatch: /e2e\/auth\.setup\.ts/ },
    { name: "chromium", testMatch: /e2e\/.*\.spec\.ts/, use: { ...devices["Desktop Chrome"], storageState: ".auth/member.json" }, dependencies: ["setup"] },
    // nightly only — see references/ci-github-actions.md
    { name: "firefox", testMatch: /e2e\/.*\.spec\.ts/, use: { ...devices["Desktop Firefox"], storageState: ".auth/member.json" }, dependencies: ["setup"] },
    { name: "webkit", testMatch: /e2e\/.*\.spec\.ts/, use: { ...devices["Desktop Safari"], storageState: ".auth/member.json" }, dependencies: ["setup"] },
    { name: "mobile", testMatch: /e2e\/.*\.spec\.ts/, use: { ...devices["Pixel 7"], storageState: ".auth/member.json" }, dependencies: ["setup"] },
  ],
});
```

Merge gate runs `--project=api --project=chromium`; nightly adds `firefox`, `webkit`, `mobile`. When the strategy flags a browser as high-risk (e.g. a large Safari audience), move it onto the gate. Retries apply once, in CI only — enough to capture a trace without hiding regressions.

## 5. API clients

One client per resource, in `support/clients/`. Two levels per operation:

- `create(payload)` → the **raw response**. API tests use it to assert status codes, error shapes, and schemas.
- `create_ok(payload)` / `createOk(payload)` → asserts 2xx, validates the schema, returns the **typed model**. Setup code (API or UI tests) uses it, so a broken setup fails loudly at the setup line.

Full client patterns, auth fixtures, and schema sourcing are in `references/api.md`.

## 6. Roles and auth

Every role in settings gets, once per session, through the API (never through the UI):

- an authenticated **API client fixture** (`member_api`, `admin_api`, plus `anon_api`);
- a **UI storage state** so browser tests start logged in.

Python uses httpx for the API clients and Playwright's request context only to produce storage state for the UI. TS uses Playwright's `request` for both. Patterns are in `references/api.md` (API auth) and `references/ui-e2e.md` (storage state).

## 7. Page objects (UI)

1. One page object per screen or significant component, in `e2e/pages/`.
2. Locators defined once, in the constructor.
3. Methods express user intent (`login(email, password)`), not mechanics.
4. A method that navigates returns the next page object.
5. Assertions live in tests, not in page objects.
6. Page objects are provided to tests by **fixtures** in both stacks — tests never construct them.

Code shapes are in `references/ui-e2e.md`.

## 8. Tagging

| Tag | Python | TS (Playwright) | Meaning |
|-----|--------|-----------------|---------|
| smoke | `@pytest.mark.smoke` | `@smoke` in title | P0, gates merge, safe on production if read-only |
| regression | `@pytest.mark.regression` | `@regression` | full set |
| writes | `@pytest.mark.writes` | `@writes` | mutates data |
| slow | `@pytest.mark.slow` | `@slow` | opt-in |

Select: `pytest -m smoke`, `npx playwright test --grep @smoke`.

## 9. Reporting

- JUnit XML from every runner into `reports/` — CI reads it.
- Human report: pytest-html (`--html=reports/report.html --self-contained-html`), Playwright HTML, Locust `--html`.
- On e2e failure: trace, screenshot, video retained.

## 10. Proof the scaffold works

The scaffold is done only when it ships **one working example test per layer in scope** and both pass Verify, including the prove-fail check:

- **API:** `I-00` — a health endpoint or a known-stable GET returns 200 and matches its schema.
- **UI:** `E-00` — the home (or landing) page loads anonymously and a key element is visible.

If Assess did not find a health endpoint or a stable page, ask the user for one. These tests prove the base URL, settings, auth wiring, browser install, and reports — and they show the conventions every later test follows.

```python
# tests/api/test_smoke.py
import pytest
from tests.support.models import Health

# I-00 health endpoint is up and matches schema
@pytest.mark.api
@pytest.mark.smoke
def test_i00_health(anon_api):
    resp = anon_api.health.get()
    assert resp.status_code == 200
    Health.model_validate(resp.json())
```

```python
# tests/e2e/test_smoke.py
import pytest
from playwright.sync_api import expect

# E-00 home page loads
@pytest.mark.e2e
@pytest.mark.smoke
@pytest.mark.role(None)
def test_e00_home_loads(home_page):
    home_page.open()
    expect(home_page.main_heading).to_be_visible()
```

```ts
// tests/api/smoke.spec.ts
import { test, expect } from "./fixtures";

test("I-00 health endpoint is up and matches schema @smoke", async ({ anonApi }) => {
  const res = await anonApi.health.get();
  expect(res.status()).toBe(200);
  await anonApi.health.getOk(); // validates schema
});
```

```ts
// tests/e2e/smoke.spec.ts
import { test, expect } from "./fixtures";

test.use({ storageState: { cookies: [], origins: [] } }); // anonymous

test("E-00 home page loads @smoke", async ({ homePage }) => {
  await homePage.open();
  await expect(homePage.mainHeading).toBeVisible();
});
```

## 11. Output of this phase

The scaffold files, the approved dependency list, the two example tests passing Verify, and the exact run command per layer recorded in `tests/TESTING.md`.
