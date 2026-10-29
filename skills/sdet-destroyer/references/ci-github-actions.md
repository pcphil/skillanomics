# CI with GitHub Actions

Loaded at `[SDET: CI]`, **on demand only**: when the user asks to wire tests into CI. Scaffold and the full pipeline create no workflows. Wires the suite into GitHub Actions so the right tests gate merges and every run leaves triageable reports. Ask the user before adding or changing a workflow that runs on push or pull request.

New workflows are separate files with a `qa-` prefix (`qa-pr.yml`, `qa-nightly.yml`, `qa-prod-smoke.yml`), so they never collide with or look like replacements for the team's CI. Add jobs into an existing workflow only when the user asks for that.

If the user opted into type checking (`references/architecture.md` §5), add a step to the Python job in `qa-pr.yml`: `uv run mypy --config-file tests/mypy.ini`. In-app repos without `.python-version`: pass `python-version` to `astral-sh/setup-uv` (match the repo's `requires-python`).

## 1. Gating model

| Trigger | Runs | Blocks merge |
|---------|------|--------------|
| pull_request / push to main | unit, integration/API, e2e `smoke` | yes |
| nightly schedule | full regression (all e2e, slow tests) | no — alerts on failure |
| manual / schedule | performance against staging | no |
| manual, with confirmation | production read-only smoke | no |

## 2. Python workflow

```yaml
# .github/workflows/qa-pr.yml
name: qa-pr
on:
  pull_request:
  push:
    branches: [main]

jobs:
  unit-integration:
    runs-on: ubuntu-latest
    services:
      postgres:                     # shared DB server; each xdist worker creates its own database
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: postgres
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 5s --health-retries 10
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5   # reads .python-version
      - run: uv sync --group test
      - run: uv run pytest tests/unit tests/integration -n auto -m "not slow" --junitxml=tests/reports/junit-unit.xml
        env:
          TEST_DATABASE_URL: postgresql://postgres:postgres@localhost:5432/postgres
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-unit
          path: tests/reports/

  api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --group test
      - run: <start the app in the background, e.g. docker compose up -d --wait>
      - run: uv run pytest tests/api -n auto --timeout 30 --junitxml=tests/reports/junit-api.xml
        env:
          TEST_TARGET_ENV: local
          TEST_BASE_URL: http://localhost:8000
          TEST_ROLE_MEMBER_USERNAME: ${{ secrets.TEST_ROLE_MEMBER_USERNAME }}
          TEST_ROLE_MEMBER_PASSWORD: ${{ secrets.TEST_ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-api
          path: tests/reports/

  e2e-smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --group test
      - run: uv run playwright install --with-deps chromium
      - run: <start the app in the background, e.g. docker compose up -d --wait>
      - run: uv run pytest tests/e2e -m smoke -n auto --reruns 1 --junitxml=tests/reports/junit-e2e.xml --html=tests/reports/e2e.html --self-contained-html
        env:
          TEST_TARGET_ENV: local
          TEST_BASE_URL: http://localhost:8000
          TEST_ROLE_MEMBER_USERNAME: ${{ secrets.TEST_ROLE_MEMBER_USERNAME }}
          TEST_ROLE_MEMBER_PASSWORD: ${{ secrets.TEST_ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-e2e
          path: |
            tests/reports/
            test-results/
```

- The test settings read `TEST_*` variables only, so CI env never collides with the app's own variables. Secrets are named the same (`TEST_ROLE_MEMBER_PASSWORD`).
- `-n`, `--timeout`, and `--reruns` live in these commands, not in global pytest config (`references/architecture.md` §2).
- Repo on poetry or pip: keep its install step (`poetry install --with test`, `pip install -r requirements-dev.txt`) and `actions/setup-python` with `python-version-file: .python-version`.

## 3. JS / TS workflow

```yaml
# .github/workflows/qa-pr.yml
name: qa-pr
on:
  pull_request:
  push:
    branches: [main]

jobs:
  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx vitest run --config tests/vitest.config.ts   # or plain, when extending an existing config
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-unit
          path: tests/reports/

  api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx playwright test -c tests/playwright.config.ts --project=api   # request-only: no browser install needed
        env:
          TEST_TARGET_ENV: local
          TEST_BASE_URL: http://localhost:3000
          TEST_ROLE_MEMBER_USERNAME: ${{ secrets.TEST_ROLE_MEMBER_USERNAME }}
          TEST_ROLE_MEMBER_PASSWORD: ${{ secrets.TEST_ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-api
          path: tests/reports/

  e2e-smoke:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1/2, 2/2]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test -c tests/playwright.config.ts --project=chromium --grep @smoke --shard ${{ matrix.shard }}
        env:
          TEST_TARGET_ENV: local
          TEST_BASE_URL: http://localhost:3000
          TEST_ROLE_MEMBER_USERNAME: ${{ secrets.TEST_ROLE_MEMBER_USERNAME }}
          TEST_ROLE_MEMBER_PASSWORD: ${{ secrets.TEST_ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-e2e-${{ strategy.job-index }}
          path: |
            tests/reports/
            test-results/
```

Start the app for e2e with Playwright's `webServer` config (`command`, `url`, `reuseExistingServer: !process.env.CI`) — simpler than a separate step.

## 4. Nightly regression and performance

```yaml
# .github/workflows/qa-nightly.yml
name: nightly
on:
  schedule:
    - cron: "0 3 * * *"
  workflow_dispatch:

jobs:
  regression:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      # setup as in qa-pr.yml; install all browsers: playwright install --with-deps
      # Python: uv run pytest -m regression -n auto --browser chromium --browser firefox --browser webkit
      #         then: uv run pytest tests/e2e -m regression -n auto --device "Pixel 7"
      # TS:     npx playwright test -c tests/playwright.config.ts   (all projects: api, chromium, firefox, webkit, mobile)
      - run: <full regression command for the stack, from above>
        env:
          TEST_TARGET_ENV: staging
          TEST_BASE_URL: ${{ vars.STAGING_URL }}
          TEST_ROLE_MEMBER_USERNAME: ${{ secrets.STAGING_ROLE_MEMBER_USERNAME }}
          TEST_ROLE_MEMBER_PASSWORD: ${{ secrets.STAGING_ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: reports-regression, path: tests/reports/ }

  perf:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install locust
      - run: locust -f tests/perf/locustfile.py --host "$TEST_BASE_URL" --headless -u 50 -r 5 -t 10m --html tests/reports/perf.html --csv tests/reports/perf
        env:
          TEST_BASE_URL: ${{ vars.STAGING_URL }}
          TEST_API_TOKEN: ${{ secrets.STAGING_API_TOKEN }}
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: reports-perf, path: tests/reports/ }
```

`STAGING_URL` is a repository variable pointing at staging — never a production URL.

**Vendor mode** (`references/vendor-apps.md`): dispatch this workflow by hand before and after each vendor upgrade window. Skip the `perf` job unless load testing is approved. Set `TEST_RUN_ID: ${{ github.run_id }}` on test steps so all workers share one record prefix, and add a sweep job:

```yaml
  sweep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v5
      - run: uv sync --group test
      - run: uv run python -m tests.support.sweep
        env:
          TEST_TARGET_ENV: staging
          TEST_BASE_URL: ${{ vars.STAGING_URL }}
          TEST_ROLE_ADMIN_USERNAME: ${{ secrets.STAGING_ROLE_ADMIN_USERNAME }}
          TEST_ROLE_ADMIN_PASSWORD: ${{ secrets.STAGING_ROLE_ADMIN_PASSWORD }}
```

## 5. Production smoke (optional)

Manual trigger only, behind a protected GitHub environment that requires reviewer approval:

```yaml
# .github/workflows/qa-prod-smoke.yml
name: prod-smoke
on: workflow_dispatch
jobs:
  smoke:
    runs-on: ubuntu-latest
    environment: production        # configure required reviewers in repo settings
    steps:
      - uses: actions/checkout@v4
      # setup + install
      - run: <smoke command, e.g. pytest -m "smoke and not writes" | npx playwright test -c tests/playwright.config.ts --grep @smoke --grep-invert @writes>
        env:
          TEST_TARGET_ENV: production
          TEST_BASE_URL: ${{ vars.PROD_URL }}
          TEST_ROLES: smoke
          TEST_ROLE_SMOKE_USERNAME: ${{ secrets.PROD_SMOKE_USERNAME }}   # dedicated read-only account
          TEST_ROLE_SMOKE_PASSWORD: ${{ secrets.PROD_SMOKE_PASSWORD }}
```

## 6. Reports

- `if: always()` on every upload so failed runs keep their reports.
- JUnit XML is the machine-readable record; to show results inline on the PR, add a JUnit reporter action the team approves.
- Playwright `test-results/` holds traces — open with `npx playwright show-trace`.

## 7. Secrets

All credentials via `secrets.*`; non-secret config via `vars.*`. Never echo secrets in run steps. Fork PRs do not receive secrets — gate e2e jobs that need them accordingly.

## 8. Adapting to other CI

The same model maps directly: GitLab CI (`stages`, `artifacts:reports:junit`, `parallel`), CircleCI (`store_test_results`, `parallelism`), Azure Pipelines (`PublishTestResults`). Keep the gating table from section 1 and translate syntax only.
