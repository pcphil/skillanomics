# Scaffold: Python

Loaded at `[SDET: Scaffold]` after `references/scaffold-plan.md`, when the suite or a layer is Python.

## 1. Layout

```
tests/                              # standalone repo: same tree; env files, .auth/, reports/ at the repo root
├── TESTING.md
├── .env.example                    # committed: every TEST_* variable, placeholder values
├── .env.local                      # committed: non-secret local values
├── .env.staging                    # committed: non-secret staging values
├── .env                            # git-ignored: secrets + personal overrides
├── .auth/                          # git-ignored: per-role storage state
├── reports/                        # git-ignored: JUnit XML, HTML reports
├── mypy.ini                        # only if the user opted in (§3)
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
│   ├── models/                     # generated from OpenAPI, or hand-written
│   │   # vendor mode adds: refresh.py, generate.py, openapi.json, openapi.overrides.json
│   └── sweep.py                    # vendor mode: delete leftover sdet- records
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
pyproject.toml                      # in-app: + test group (+ pytest markers if no config). Standalone: full sample (§3)
.python-version                     # standalone only
```

There is **no `.env.production`**. Production values exist only in CI (`references/scaffold-plan.md` §3).

Give every test file a basename unique across the whole suite, prefixed by layer: `test_api_orders.py`, `test_e2e_orders.py`. The test folders have no `__init__.py`, and pytest's default import mode stops with "import file mismatch" when two folders hold the same file name.

## 2. Settings (python-dotenv + pydantic-settings)

Load every file by **explicit path** built from `settings.py`'s own location. A bare `load_dotenv()` searches upward for `.env` and, when `tests/.env` is missing, loads the app's root `.env` into the test process — never call it without a path.

Settings load **lazily**: nothing reads env files at import time, so unit tests run on a fresh clone with no `.env`. Each consumer requires only the values it needs, and a missing value fails with the exact variable name.

```python
# tests/settings.py
import os
from functools import lru_cache
from pathlib import Path
from typing import Literal
from urllib.parse import urlparse
from dotenv import load_dotenv
from pydantic import BaseModel
from pydantic_settings import BaseSettings, SettingsConfigDict

ENV_DIR = Path(__file__).parent  # standalone repo: Path(__file__).parents[1]

class Role(BaseModel):
    username: str
    password: str
    totp_secret: str | None = None  # vendor SSO with MFA (references/vendor-apps.md §4)

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="TEST_")

    target_env: Literal["local", "staging", "production"] = "local"
    base_url: str = ""
    api_url: str = ""
    database_url: str = ""
    roles: str = "member"  # comma-separated role names
    default_role: str = "member"  # role for tests with no role marker; production smoke uses "smoke"
    login_path: str = "/auth/login"  # the app's API login endpoint, from Assess
    production_hosts: str = ""  # comma-separated hostnames that are production (guard below)

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
        return Role(username=user, password=pwd, totp_secret=os.environ.get(f"TEST_ROLE_{key}_TOTP_SECRET"))

def load_env_files() -> str:
    """Exported vars win, then .env, then .env.<target>. Returns the target env."""
    load_dotenv(ENV_DIR / ".env", override=False)
    target = os.environ.get("TEST_TARGET_ENV", "local")
    load_dotenv(ENV_DIR / f".env.{target}", override=False)
    return target

@lru_cache
def get_settings() -> Settings:
    load_env_files()
    settings = Settings()
    _guard_production(settings)
    return settings

def _guard_production(s: Settings) -> None:
    """A production host labelled as another env fails before any test runs."""
    prod_hosts = {h.strip() for h in s.production_hosts.split(",") if h.strip()}
    for url in (s.base_url, s.api_url):
        if url and urlparse(url).hostname in prod_hosts and s.target_env != "production":
            raise RuntimeError(f"{url} is a production host but TEST_TARGET_ENV={s.target_env}")
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
TEST_DEFAULT_ROLE=member
TEST_LOGIN_PATH=/auth/login
TEST_PRODUCTION_HOSTS=app.example.com
TEST_ROLE_MEMBER_USERNAME=member@example.test
TEST_ROLE_MEMBER_PASSWORD=change-me
TEST_ROLE_ADMIN_USERNAME=admin@example.test
TEST_ROLE_ADMIN_PASSWORD=change-me
```

## 3. Project config

### In-app: pyproject.toml additions

```toml
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
  "psycopg[binary]",          # Postgres driver; use the app's own driver if it has one
  "locust",                   # perf in scope
]

# only when the repo has no pytest config (`references/scaffold-plan.md` §2 rule 3); otherwise add the markers to the existing one
[tool.pytest.ini_options]
pythonpath = ["."]
markers = [
  "smoke: P0 critical path, gates every merge",
  "regression: full coverage, nightly / pre-release",
  "api: API layer",
  "e2e: browser layer",
  "writes: creates or mutates data; never runs on production",
  "slow: long-running, opt-in",
  "role(name): run the e2e test as this role; role(None) = anonymous",
]
```

No `addopts`, `testpaths`, or `timeout`: a bare `pytest` keeps collecting and behaving exactly as before. Run options go on the commands (§5).

### Standalone: pyproject.toml

The same test group and markers, plus settings the repo owns outright:

```toml
[project]
requires-python = ">=3.11"

[tool.pytest.ini_options]
pythonpath = ["."]
testpaths = ["tests"]
addopts = ["--strict-markers"]
timeout = 60
markers = [ ... as above ... ]
```

And a `.python-version` with `3.12`, so uv, pyenv, and CI use the same interpreter.

- Install: `uv sync --group test`; commit `uv.lock`. Repo on poetry/pip-tools/pip → same packages in its format.
- `-n auto`, `--reruns`, and report paths are never in config — they go in run commands (§5).

### Type checking (opt-in)

Ask in Scaffold, with the dependency list: "Add type checking for the test support layer? (mypy strict on `tests/support/` and `settings.py`, its own command; your existing mypy/ruff setup is untouched.) Default: no." Skip the question when the repo already type-checks `tests/`. Record the answer in `TESTING.md` so later runs do not ask again.

On yes, add `mypy` to the test group and:

```ini
# tests/mypy.ini — run: uv run mypy --config-file tests/mypy.ini
[mypy]
files = tests/support, tests/settings.py
strict = true
plugins = pydantic.mypy
```

Standalone repo: the same settings as `[tool.mypy]` in its `pyproject.toml`.

## 4. Fixture modules

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

## 5. Commands

Python layer commands (record them in `TESTING.md`):

```bash
uv run pytest tests/unit -n auto --junitxml=tests/reports/junit-unit.xml
uv run pytest tests/api -n auto --timeout 30 --junitxml=tests/reports/junit-api.xml
uv run pytest tests/e2e -m smoke -n auto --junitxml=tests/reports/junit-e2e.xml --html=tests/reports/e2e.html --self-contained-html   # CI adds: --reruns 1
uv run pytest tests/e2e --browser firefox --browser webkit   # nightly / pre-release
uv run pytest tests/e2e --device "Pixel 7"                     # nightly / pre-release
```

When the repo has a task runner (Makefile, justfile, `package.json` scripts, tox, nox), add these as targets there (`test-unit`, `test-api`, `test-e2e`, `test-smoke`) as part of the footprint preview. Otherwise record them in `TESTING.md` — in an app repo, task runners stay the team's choice.

pytest-playwright runs Chromium by default. Its base URL comes from the `base_url` override in `tests/e2e/conftest.py`:

```python
# tests/e2e/conftest.py (excerpt)
import pytest

@pytest.fixture(scope="session")
def base_url(env_settings):
    return env_settings.require("base_url")
```

## 6. Proof tests

```python
# tests/api/test_api_smoke.py
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
# tests/e2e/test_e2e_smoke.py
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
