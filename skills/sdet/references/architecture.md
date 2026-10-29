# Framework architecture

Loaded at `[SDET: Scaffold]`, when the repo has no test framework. These are the structural choices that decide whether a suite scales or rots. Size them to the agreed strategy — scaffold only the layers it calls for.

List every dependency to add and ask the user before installing.

## 1. Layout

### Python

```
tests/
├── TESTING.md
├── conftest.py             # root fixtures: settings, base_url, api client
├── config.py               # env-driven settings (BASE_URL, credentials)
├── factories/              # data factories (see test-data.md)
├── unit/
├── api/
│   └── schemas/            # JSON schemas or pydantic models for responses
├── e2e/
│   ├── conftest.py         # browser/auth fixtures
│   └── pages/              # page objects
└── perf/
    └── locustfile.py
pytest.ini                  # or [tool.pytest.ini_options] in pyproject.toml
.env.example
```

### JS / TS

```
tests/
├── TESTING.md
├── config.ts               # env-driven settings
├── factories/
├── unit/                   # *.test.ts (Vitest)
├── api/                    # *.spec.ts (Playwright request) or *.test.ts (Vitest)
│   └── schemas/
├── e2e/
│   ├── fixtures.ts         # extended test with page objects + auth
│   └── pages/
└── perf/
    └── locustfile.py       # Locust is Python on every stack
vitest.config.ts
playwright.config.ts
.env.example
```

When the repo already co-locates tests next to source (`src/**/*.test.ts`), keep unit tests there and put only api/e2e/perf under `tests/`.

## 2. Env-driven config

One place reads the environment; tests import from it. No URL or secret appears in a test file.

```python
# tests/config.py
import os

BASE_URL = os.environ.get("BASE_URL", "http://localhost:8000")
API_TOKEN = os.environ.get("API_TOKEN", "")
TARGET_ENV = os.environ.get("TARGET_ENV", "local")  # local | staging | production
```

```ts
// tests/config.ts
export const BASE_URL = process.env.BASE_URL ?? "http://localhost:3000";
export const API_TOKEN = process.env.API_TOKEN ?? "";
export const TARGET_ENV = process.env.TARGET_ENV ?? "local";
```

`.env.example` lists every variable with a placeholder value. Real `.env` stays git-ignored — add it to `.gitignore` if missing.

### Production guard

Mark tests that write data so they cannot run against production:

```python
# tests/conftest.py
import pytest
from tests.config import TARGET_ENV

def pytest_collection_modifyitems(config, items):
    if TARGET_ENV != "production":
        return
    skip = pytest.mark.skip(reason="production run: read-only smoke only")
    for item in items:
        if "smoke" not in item.keywords or "writes" in item.keywords:
            item.add_marker(skip)
```

```ts
// playwright.config.ts (excerpt)
grep: process.env.TARGET_ENV === "production" ? /@smoke/ : undefined,
grepInvert: process.env.TARGET_ENV === "production" ? /@writes/ : undefined,
```

## 3. Runner config

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
```

`--strict-markers` turns a mistyped marker into an error instead of a silent no-op.

### Vitest

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

### Playwright Test

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./tests",
  testMatch: ["e2e/**/*.spec.ts", "api/**/*.spec.ts"],
  fullyParallel: true,
  retries: process.env.CI ? 1 : 0,
  reporter: [["list"], ["junit", { outputFile: "reports/junit-e2e.xml" }], ["html", { open: "never", outputFolder: "reports/html" }]],
  use: {
    baseURL: process.env.BASE_URL ?? "http://localhost:3000",
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  projects: [{ name: "chromium", use: { ...devices["Desktop Chrome"] } }],
});
```

Retries apply to e2e only, once, in CI — enough to collect a trace without hiding regressions.

## 4. Fixtures

- Root fixtures provide settings and clients; layer fixtures (e2e, api) build on them.
- Scope expensive setup (containers, auth state) to `session`; scope data to `function`.
- A fixture that creates data also deletes it (yield/teardown).

```python
# tests/conftest.py
import httpx
import pytest
from tests.config import BASE_URL, API_TOKEN

@pytest.fixture(scope="session")
def api():
    with httpx.Client(base_url=BASE_URL, headers={"Authorization": f"Bearer {API_TOKEN}"}, timeout=10) as client:
        yield client
```

## 5. Page objects (UI only)

1. One page object per screen or significant component.
2. Locators defined once, in the constructor.
3. Methods express user intent (`login(email, password)`), not mechanics.
4. A method that navigates returns the next page object.
5. Assertions live in tests, not in page objects.
6. A `BasePage` holds shared helpers.

Code shapes for both stacks are in `references/ui-e2e.md`.

## 6. Tagging

| Tag | Python | TS (Playwright) | Meaning |
|-----|--------|-----------------|---------|
| smoke | `@pytest.mark.smoke` | `@smoke` in title or `tag` option | P0, gates merge, safe for prod if read-only |
| regression | `@pytest.mark.regression` | `@regression` | full set |
| writes | `@pytest.mark.writes` | `@writes` | mutates data |
| slow | `@pytest.mark.slow` | `@slow` | opt-in |

Select: `pytest -m smoke`, `npx playwright test --grep @smoke`.

## 7. Reporting

- JUnit XML from every runner into `reports/` — CI reads it.
- Human report: pytest-html (`--html=reports/report.html --self-contained-html`), Playwright HTML, Locust `--html`.
- On e2e failure: trace, screenshot, video retained.
- Add `reports/` to `.gitignore`.

## 8. Output of this phase

The scaffold files, the dependency list (approved by the user before install), and a one-command run per layer that executes — even with zero tests — without errors.
