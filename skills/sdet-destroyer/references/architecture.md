# Framework architecture

Loaded at `[SDET: Scaffold]`, when the repo has no test framework for the layers in scope. These are the structural choices that decide whether a suite scales or rots. Size them to the agreed strategy — scaffold only the layers it calls for.

List every dependency to add and ask the user before installing.

## 1. Where the suite lives

| Situation | Location | Stack |
|-----------|----------|-------|
| One service, tests ship with the code (default) | `tests/` inside the app repo | the app's stack |
| Several services (multiple app services in compose, a monorepo with several apps), or the user asks | a standalone test repo, same tree at its root | Python by default; confirm with the user |

Unit tests stay in the app repo either way. Only API and UI move to a standalone repo. Record the decision in the strategy and in `tests/TESTING.md`.

## 2. Coexisting with an existing repo

An in-app scaffold adds to a repo the team already owns. Adding new tests must never change how existing tests or the app behave.

1. **Shared files are merged, never overwritten** — `pyproject.toml`, root `conftest.py`, `.gitignore`, CI workflows, `.pre-commit-config.yaml`, `package.json`. Add the smallest change in the format the repo already uses, show the diff, and wait for approval.
2. **One pytest config.** If any of `pytest.ini`, `pyproject.toml [tool.pytest.ini_options]`, `setup.cfg [tool:pytest]`, or `tox.ini [pytest]` exists, extend that file. Create `[tool.pytest.ini_options]` only when none exists — `pytest.ini` silently wins over `pyproject.toml`, so two configs means one is ignored.
3. **Follow the repo's tooling.** Dependency tool (uv, poetry, pip-tools, plain pip), ruff/mypy config, and `.python-version` stay as the team set them; add a test group in their format (e.g. `[tool.poetry.group.test.dependencies]`).
4. **Scope stricter options to new tests.** `-n auto`, `--timeout`, and `--reruns` go in the layer run commands and CI, never in global `addopts`. Add `--strict-markers` globally only after registering every marker the existing suite uses.
5. **Avoid fixture collisions.** In Assess, list existing fixtures (`pytest --fixtures`). Known clash: pytest-django provides `settings` — this skill's settings fixture is `env_settings`. Rename any other fixture that collides.
6. **Leave existing tests in place.** Do not move or rename them. Add `api/`, `e2e/`, `support/` beside them. If a root `tests/conftest.py` exists, append only the `pytest_plugins` line. When existing unit tests live flat in `tests/`, new unit tests follow that location; record it in `TESTING.md`.
7. **Env files stay inside `tests/`** (section 4) so no framework loader (Next.js, Vite, Django, docker compose) picks them up.

## 3. Layout

### Python

```
tests/                              # standalone repo: this tree sits at the repo root
├── TESTING.md
├── .env.example                    # committed: every TEST_* variable, placeholder values
├── .env.local                      # committed: non-secret local values
├── .env.staging                    # committed: non-secret staging values
├── .env                            # git-ignored: secrets + personal overrides
├── conftest.py                     # pytest_plugins only (+ nothing else)
├── settings.py                     # env loading + typed settings
├── support/
│   ├── fixtures/
│   │   ├── settings.py             # env_settings fixture, production guard hook
│   │   ├── api.py                  # api_as, anon_api, <role>_api
│   │   ├── auth.py                 # storage_state_for (UI)
│   │   └── data.py                 # database_url, db_session, data fixtures
│   ├── clients/                    # one API client per resource (references/api.md)
│   │   └── <service>/              # standalone multi-service: group by service
│   └── models/                     # generated from OpenAPI, or hand-written
├── factories/                      # data builders (references/test-data.md)
├── unit/
│   └── test_env_example.py         # U-00: .env.example stays complete
├── integration/                    # real DB / framework test client (references/unit-integration.md)
├── api/
│   └── <service>/                  # standalone multi-service: group by service
├── e2e/
│   ├── conftest.py                 # base_url, browser_context_args, page-object fixtures
│   └── pages/
└── perf/
    └── locustfile.py
pyproject.toml                      # pytest, test dependency group, ruff, mypy
.python-version
```

There is **no `.env.production`**. Production values exist only in CI (section 4).

### TS

```
tests/
├── TESTING.md
├── .env.example  .env.local  .env.staging  .env   # same convention as Python
├── settings.ts
├── support/{clients/,models/}
├── factories/
├── unit/                           # *.test.ts (Vitest); or co-located src/**/*.test.ts
├── api/
│   └── fixtures.ts                 # role-based API fixtures
├── e2e/
│   ├── fixtures.ts                 # page objects, extends api fixtures
│   ├── auth.setup.ts               # per-role storage state
│   └── pages/
└── perf/locustfile.py              # Locust is Python on every stack
playwright.config.ts
vitest.config.ts
```

Add to `.gitignore` (merge): `reports/`, `.auth/`, `tests/.env`. Standalone repo: `.env` at root instead of `tests/.env`.

## 4. Env files and settings

### Convention

- **Every test variable starts with `TEST_`** — `TEST_TARGET_ENV`, `TEST_BASE_URL`, `TEST_API_URL`, `TEST_DATABASE_URL`, `TEST_ROLES`, `TEST_ROLE_<NAME>_USERNAME` / `_PASSWORD`. Nothing collides with the app's own variables (Vite's built-in `BASE_URL`, an app `BASE_URL`, `DATABASE_URL`).
- **Files live in `ENV_DIR`** — `tests/` in-app, the repo root in a standalone repo.
- **Precedence:** exported variables (shell, CI) → `.env` (secrets, personal overrides) → `.env.<TEST_TARGET_ENV>` (committed, non-secret). `TEST_TARGET_ENV` may itself be set in `.env`, defaulting to `local`.
- **Secrets** live only in the git-ignored `.env` or in CI secrets. Committed files hold URLs and non-secret values.
- **Production** has no env file. A production run requires exporting its values deliberately; CI supplies them from secrets behind a protected environment (`references/ci-github-actions.md` §5).

### Python (python-dotenv + pydantic-settings)

Load every file by **explicit path** built from `settings.py`'s own location. A bare `load_dotenv()` searches upward for `.env` and, when `tests/.env` is missing, loads the app's root `.env` into the test process — never call it without a path.

Settings load **lazily**: nothing reads env files at import time, so unit tests run on a fresh clone with no `.env`. Each consumer requires only the values it needs, and a missing value fails with the exact variable name.

```python
# tests/settings.py
import os
from functools import lru_cache
from pathlib import Path
from typing import Literal
from dotenv import load_dotenv
from pydantic import BaseModel
from pydantic_settings import BaseSettings, SettingsConfigDict

ENV_DIR = Path(__file__).parent  # standalone repo: Path(__file__).parents[1]

class Role(BaseModel):
    username: str
    password: str

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="TEST_")

    target_env: Literal["local", "staging", "production"] = "local"
    base_url: str = ""
    api_url: str = ""
    database_url: str = ""
    roles: str = "member"  # comma-separated role names

    def require(self, field: str) -> str:
        value = getattr(self, field)
        if not value:
            raise RuntimeError(
                f"TEST_{field.upper()} missing for {self.target_env} — set it in "
                f"{ENV_DIR / '.env'} or {ENV_DIR / f'.env.{self.target_env}'} (see .env.example)"
            )
        return value

    @property
    def api_base(self) -> str:
        return self.api_url or self.require("base_url")

    @property
    def role_names(self) -> list[str]:
        return [r.strip() for r in self.roles.split(",") if r.strip()]

    def role(self, name: str) -> Role:
        key = name.upper()
        user = os.environ.get(f"TEST_ROLE_{key}_USERNAME")
        pwd = os.environ.get(f"TEST_ROLE_{key}_PASSWORD")
        if not user or not pwd:
            raise RuntimeError(f"TEST_ROLE_{key}_USERNAME / TEST_ROLE_{key}_PASSWORD missing for {self.target_env}")
        return Role(username=user, password=pwd)

def load_env_files() -> str:
    """Exported vars win, then .env, then .env.<target>. Returns the target env."""
    load_dotenv(ENV_DIR / ".env", override=False)
    target = os.environ.get("TEST_TARGET_ENV", "local")
    load_dotenv(ENV_DIR / f".env.{target}", override=False)
    return target

@lru_cache
def get_settings() -> Settings:
    load_env_files()
    return Settings()
```

```python
# tests/support/fixtures/settings.py
import pytest
from tests.settings import Settings, get_settings, load_env_files

@pytest.fixture(scope="session")
def env_settings() -> Settings:
    return get_settings()

def pytest_collection_modifyitems(config, items):
    """Production guard: on production, only read-only smoke tests run."""
    if load_env_files() != "production":
        return
    skip = pytest.mark.skip(reason="production run: read-only smoke only")
    for item in items:
        if "smoke" not in item.keywords or "writes" in item.keywords:
            item.add_marker(skip)
```

```python
# tests/unit/test_env_example.py
from dotenv import dotenv_values
from tests.settings import ENV_DIR, Settings

# U-00 .env.example documents every test setting
def test_u00_env_example_complete():
    example = dotenv_values(ENV_DIR / ".env.example")
    expected = {f"TEST_{name.upper()}" for name in Settings.model_fields}
    for role in (example.get("TEST_ROLES") or "member").split(","):
        key = role.strip().upper()
        expected |= {f"TEST_ROLE_{key}_USERNAME", f"TEST_ROLE_{key}_PASSWORD"}
    missing = expected - example.keys()
    assert not missing, f".env.example is missing: {sorted(missing)}"
```

`.env.example`:

```bash
# Copy to .env for secrets and personal overrides. Committed .env.<env> files hold non-secret values.
TEST_TARGET_ENV=local
TEST_BASE_URL=http://localhost:8000
TEST_API_URL=
TEST_DATABASE_URL=
TEST_ROLES=member,admin
TEST_ROLE_MEMBER_USERNAME=member@example.test
TEST_ROLE_MEMBER_PASSWORD=change-me
TEST_ROLE_ADMIN_USERNAME=admin@example.test
TEST_ROLE_ADMIN_PASSWORD=change-me
```

### TS (dotenv + zod)

Same files, same `TEST_` names, same precedence. TS settings are validated when first imported.

```ts
// tests/settings.ts
import path from "node:path";
import dotenv from "dotenv";
import { z } from "zod";

const envDir = path.resolve("tests"); // standalone repo: path.resolve(".")
dotenv.config({ path: path.join(envDir, ".env") }); // never overrides exported vars
const targetEnv = process.env.TEST_TARGET_ENV ?? "local";
dotenv.config({ path: path.join(envDir, `.env.${targetEnv}`) });

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} missing for ${targetEnv} — see tests/.env.example`);
  return value;
}

const env = z
  .object({
    TEST_TARGET_ENV: z.enum(["local", "staging", "production"]),
    TEST_BASE_URL: z.string().url(),
    TEST_API_URL: z.string().url().optional(),
    TEST_ROLES: z.string().default("member"),
  })
  .parse({ ...process.env, TEST_TARGET_ENV: targetEnv });

export const roleNames = env.TEST_ROLES.split(",").map((r) => r.trim()).filter(Boolean);
export const roles = Object.fromEntries(
  roleNames.map((name) => {
    const key = name.toUpperCase();
    return [name, { username: required(`TEST_ROLE_${key}_USERNAME`), password: required(`TEST_ROLE_${key}_PASSWORD`) }];
  }),
) as Record<string, { username: string; password: string }>;

export const settings = {
  targetEnv: env.TEST_TARGET_ENV,
  baseUrl: env.TEST_BASE_URL,
  apiBase: env.TEST_API_URL ?? env.TEST_BASE_URL,
};
```

TS production guard:

```ts
// playwright.config.ts (excerpt)
grep: settings.targetEnv === "production" ? /@smoke/ : undefined,
grepInvert: settings.targetEnv === "production" ? /@writes/ : undefined,
```

## 5. Python project config

### pyproject.toml (new scaffold; in-app: merge these sections only)

```toml
[project]
requires-python = ">=3.11"

[dependency-groups]
test = [
  "pytest>=8",
  "pytest-xdist",
  "pytest-timeout",
  "pytest-html",
  "pytest-rerunfailures",
  "pytest-mock",
  "pytest-playwright",        # UI in scope
  "httpx",
  "pydantic>=2",
  "pydantic-settings",
  "python-dotenv",
  "faker",
  "factory-boy",
  "testcontainers[postgres]", # integration needs a real DB
  "sqlalchemy",               # per-worker database setup
  "ruff",
  "mypy",
]

[tool.pytest.ini_options]
pythonpath = ["."]
testpaths = ["tests"]
addopts = ["--strict-markers", "--junitxml=reports/junit.xml"]
timeout = 60
markers = [
  "smoke: P0 critical path, gates every merge",
  "regression: full coverage, nightly / pre-release",
  "api: API layer",
  "e2e: browser layer",
  "writes: creates or mutates data; never runs on production",
  "slow: long-running, opt-in",
  "role(name): run the e2e test as this role; role(None) = anonymous",
]

[tool.ruff.lint]
extend-select = ["PT", "B", "I"]

[tool.mypy]
files = ["tests/support", "tests/settings.py"]
strict = true
```

- Install: `uv sync --group test`; commit `uv.lock`. Repo on poetry/pip-tools/pip → same packages in its format.
- `timeout = 60` and `--strict-markers` sit in global config **only for a new scaffold**. In-app with existing tests, follow section 2 rule 4.
- `-n auto` and `--reruns` are never global — they go in run commands (section 7).
- mypy covers typed infrastructure (`support/`, settings), not test bodies.

### .python-version

```
3.12
```

CI, uv, and pyenv read it, so local and CI cannot drift. In-app: keep the repo's existing file.

## 6. Fixture modules

Fixtures live in topic modules under `tests/support/fixtures/`, registered once:

```python
# tests/conftest.py
pytest_plugins = [
    "tests.support.fixtures.settings",
    "tests.support.fixtures.api",
    "tests.support.fixtures.auth",
    "tests.support.fixtures.data",
]
```

- `pytest_plugins` goes only in `tests/conftest.py` — pytest rejects it in nested conftests.
- Fixtures that **override a plugin's fixture** (`base_url`, `browser_context_args` from pytest-playwright) go in `tests/e2e/conftest.py`, where conftest precedence over plugins is guaranteed.
- Layer conftests hold only layer-specific fixtures (page objects, overrides). `pytest --fixtures` shows where each fixture comes from.
- Session-scoped fixtures run once **per xdist worker**. That is intended for logins and storage state (isolated per worker). Heavy resources such as databases follow `references/test-data.md` §3.

## 7. Runners and commands

| Stack | Unit | API | UI |
|-------|------|-----|----|
| Python | pytest | pytest + httpx | pytest + pytest-playwright (sync API) |
| TS | Vitest | Playwright Test (`request`) | Playwright Test |

When the repo already uses another runner for a layer, keep it.

Python layer commands (record them in `TESTING.md`):

```bash
uv run pytest tests/unit -n auto
uv run pytest tests/api -n auto --timeout 30
uv run pytest tests/e2e -m smoke -n auto            # CI adds: --reruns 1
uv run pytest tests/e2e --browser firefox --browser webkit   # nightly
uv run pytest tests/e2e --device "Pixel 7"                     # nightly
```

pytest-playwright runs Chromium by default. Its base URL comes from the `base_url` override in `tests/e2e/conftest.py`:

```python
# tests/e2e/conftest.py (excerpt)
import pytest

@pytest.fixture(scope="session")
def base_url(env_settings):
    return env_settings.require("base_url")
```

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

const e2e = (name: string, device: object) => ({
  name,
  testMatch: /e2e\/.*\.spec\.ts/,
  use: { ...device, storageState: ".auth/member.json" },
  dependencies: ["setup"],
});

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
    e2e("chromium", devices["Desktop Chrome"]),
    // nightly only — references/ci-github-actions.md
    e2e("firefox", devices["Desktop Firefox"]),
    e2e("webkit", devices["Desktop Safari"]),
    e2e("mobile", devices["Pixel 7"]),
  ],
});
```

Merge gate runs `--project=api --project=chromium`; nightly adds `firefox`, `webkit`, `mobile`. When the strategy flags a browser as high-risk (e.g. a large Safari audience), move it onto the gate. Retries apply once, in CI only.

## 8. API clients

One client per resource in `support/clients/`, two levels per operation:

- `create(payload)` → the **raw response**, for asserting status codes, error shapes, schemas.
- `create_ok(payload)` / `createOk(payload)` → asserts 2xx, validates the schema, returns the **typed model**, for setup.

Patterns, role fixtures, and schema sourcing: `references/api.md`.

## 9. Roles and auth

Every role in `TEST_ROLES` gets, once per session (per worker under xdist), through the API — never through the UI:

- an authenticated **API client fixture** (`member_api`, `admin_api`, plus `anon_api`);
- a **UI storage state** so browser tests start logged in.

Python: httpx for API clients; Playwright's request context only to produce storage state. TS: Playwright `request` for both. Patterns: `references/api.md` §4, `references/ui-e2e.md` §5.

## 10. Page objects (UI)

1. One page object per screen or significant component, in `e2e/pages/`.
2. Locators defined once, in the constructor.
3. Methods express user intent (`login(email, password)`), not mechanics.
4. A method that navigates returns the next page object.
5. Assertions live in tests, not in page objects.
6. Page objects reach tests through **fixtures** in both stacks — tests never construct them.

Code shapes: `references/ui-e2e.md`.

## 11. Tagging

| Tag | Python | TS (Playwright) | Meaning |
|-----|--------|-----------------|---------|
| smoke | `@pytest.mark.smoke` | `@smoke` in title | P0, gates merge, safe on production if read-only |
| regression | `@pytest.mark.regression` | `@regression` | full set |
| writes | `@pytest.mark.writes` | `@writes` | mutates data |
| slow | `@pytest.mark.slow` | `@slow` | opt-in |

## 12. Reporting

- JUnit XML from every runner into `reports/` — CI reads it.
- Human report: pytest-html (`--html=reports/report.html --self-contained-html`), Playwright HTML, Locust `--html`.
- On e2e failure: trace, screenshot, video retained.

## 13. Proof the scaffold works

The scaffold is done only when it ships **one working example test per layer in scope**, plus U-00, and they pass Verify — including the prove-fail check:

- **U-00** — `.env.example` documents every setting (section 4). Runs with no `.env`.
- **I-00** — a health endpoint or known-stable GET returns 200 and matches its schema.
- **E-00** — the home (or landing) page loads anonymously and a key element is visible.

If Assess found no health endpoint or stable page, ask the user for one.

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

## 14. Output of this phase

The scaffold files, the approved dependency list and shared-file diffs, U-00 / I-00 / E-00 passing Verify, and the exact run command per layer recorded in `tests/TESTING.md`.
