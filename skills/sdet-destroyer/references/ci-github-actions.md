# CI with GitHub Actions

Loaded at `[SDET: CI]`. Wires the suite into GitHub Actions so the right tests gate merges and every run leaves triageable reports. Ask the user before adding or changing a workflow that runs on push or pull request.

When the repo already has workflows, add jobs to the existing test workflow in its style rather than creating a parallel one.

## 1. Gating model

| Trigger | Runs | Blocks merge |
|---------|------|--------------|
| pull_request / push to main | unit, integration/API, e2e `smoke` | yes |
| nightly schedule | full regression (all e2e, slow tests) | no — alerts on failure |
| manual / schedule | performance against staging | no |
| manual, with confirmation | production read-only smoke | no |

## 2. Python workflow

```yaml
# .github/workflows/tests.yml
name: tests
on:
  pull_request:
  push:
    branches: [main]

jobs:
  unit-api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/unit tests/api -m "not slow" --junitxml=reports/junit-unit-api.xml
        env:
          TARGET_ENV: local
          BASE_URL: http://localhost:8000
          ROLE_MEMBER_USERNAME: ${{ secrets.ROLE_MEMBER_USERNAME }}
          ROLE_MEMBER_PASSWORD: ${{ secrets.ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-unit-api
          path: reports/

  e2e-smoke:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
          cache: pip
      - run: pip install -r requirements-dev.txt
      - run: playwright install --with-deps chromium
      - run: <start the app in the background, e.g. docker compose up -d --wait>
      - run: pytest tests/e2e -m smoke --base-url "$BASE_URL" --junitxml=reports/junit-e2e.xml --html=reports/e2e.html --self-contained-html
        env:
          TARGET_ENV: local
          BASE_URL: http://localhost:8000
          ROLE_MEMBER_USERNAME: ${{ secrets.ROLE_MEMBER_USERNAME }}
          ROLE_MEMBER_PASSWORD: ${{ secrets.ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-e2e
          path: |
            reports/
            test-results/
```

The `unit-api` job needs the service running when `tests/api` hits a live server — add the same start step as `e2e-smoke`, or split API tests into their own job.

## 3. JS / TS workflow

```yaml
# .github/workflows/tests.yml
name: tests
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
      - run: npx vitest run
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-unit
          path: reports/

  api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: npm
      - run: npm ci
      - run: npx playwright test --project=api   # request-only: no browser install needed
        env:
          TARGET_ENV: local
          BASE_URL: http://localhost:3000
          ROLE_MEMBER_USERNAME: ${{ secrets.ROLE_MEMBER_USERNAME }}
          ROLE_MEMBER_PASSWORD: ${{ secrets.ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-api
          path: reports/

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
      - run: npx playwright test --project=chromium --grep @smoke --shard ${{ matrix.shard }}
        env:
          TARGET_ENV: local
          BASE_URL: http://localhost:3000
          ROLE_MEMBER_USERNAME: ${{ secrets.ROLE_MEMBER_USERNAME }}
          ROLE_MEMBER_PASSWORD: ${{ secrets.ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: reports-e2e-${{ strategy.job-index }}
          path: |
            reports/
            test-results/
```

Start the app for e2e with Playwright's `webServer` config (`command`, `url`, `reuseExistingServer: !process.env.CI`) — simpler than a separate step.

## 4. Nightly regression and performance

```yaml
# .github/workflows/nightly.yml
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
      # setup as in tests.yml; install all browsers: playwright install --with-deps
      # Python: pytest -m regression --browser chromium --browser firefox --browser webkit
      #         then: pytest tests/e2e -m regression --device "Pixel 7"
      # TS:     npx playwright test   (all projects: api, chromium, firefox, webkit, mobile)
      - run: <full regression command for the stack, from above>
        env:
          TARGET_ENV: staging
          BASE_URL: ${{ vars.STAGING_URL }}
          ROLE_MEMBER_USERNAME: ${{ secrets.STAGING_ROLE_MEMBER_USERNAME }}
          ROLE_MEMBER_PASSWORD: ${{ secrets.STAGING_ROLE_MEMBER_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: reports-regression, path: reports/ }

  perf:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: "3.12" }
      - run: pip install locust
      - run: locust -f tests/perf/locustfile.py --host "$BASE_URL" --headless -u 50 -r 5 -t 10m --html reports/perf.html --csv reports/perf
        env:
          BASE_URL: ${{ vars.STAGING_URL }}
          API_TOKEN: ${{ secrets.STAGING_API_TOKEN }}
      - uses: actions/upload-artifact@v4
        if: always()
        with: { name: reports-perf, path: reports/ }
```

`STAGING_URL` is a repository variable pointing at staging — never a production URL.

## 5. Production smoke (optional)

Manual trigger only, behind a protected GitHub environment that requires reviewer approval:

```yaml
# .github/workflows/prod-smoke.yml
name: prod-smoke
on: workflow_dispatch
jobs:
  smoke:
    runs-on: ubuntu-latest
    environment: production        # configure required reviewers in repo settings
    steps:
      - uses: actions/checkout@v4
      # setup + install
      - run: <smoke command, e.g. pytest -m "smoke and not writes" | npx playwright test --grep @smoke --grep-invert @writes>
        env:
          TARGET_ENV: production
          BASE_URL: ${{ vars.PROD_URL }}
```

## 6. Reports

- `if: always()` on every upload so failed runs keep their reports.
- JUnit XML is the machine-readable record; to show results inline on the PR, add a JUnit reporter action the team approves.
- Playwright `test-results/` holds traces — open with `npx playwright show-trace`.

## 7. Secrets

All credentials via `secrets.*`; non-secret config via `vars.*`. Never echo secrets in run steps. Fork PRs do not receive secrets — gate e2e jobs that need them accordingly.

## 8. Adapting to other CI

The same model maps directly: GitLab CI (`stages`, `artifacts:reports:junit`, `parallel`), CircleCI (`store_test_results`, `parallelism`), Azure Pipelines (`PublishTestResults`). Keep the gating table from section 1 and translate syntax only.
