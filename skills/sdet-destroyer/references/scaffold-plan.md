# Scaffold plan

Loaded at `[SDET: Scaffold]`, when the repo has no test framework for the layers in scope. These are the structural choices that decide whether a suite scales or rots. Size them to the agreed strategy — scaffold only the layers it calls for.

After this file, load the stack file for each stack Assess found — `references/scaffold-python.md`, `references/scaffold-ts.md`, or both for a mixed repo.

List every dependency to add and ask the user before installing.

## 1. Where the suite lives

| Situation | Location | Stack |
|-----------|----------|-------|
| One service, tests ship with the code (default) | `tests/` inside the app repo | the app's stack |
| Several services (multiple app services in compose, a monorepo with several apps), a vendor product (`references/vendor-apps.md`), or the user asks | a standalone test repo: same `tests/` tree; configs, env files, `.auth/`, `reports/` at its root | Python by default; confirm with the user |

Unit tests stay in the app repo either way. Only API and UI move to a standalone repo. Record the decision in the strategy and in `tests/TESTING.md`.

**Root footprint.** An app repo already has owners: in-app, everything the scaffold adds lives inside `tests/` — runner configs, auth state, reports included — and the repo root gets only the additions in §2 rule 1. A standalone repo is new, so its configs sit at its root.

## 2. Coexisting with an existing repo

An in-app scaffold adds to a repo the team already owns. Adding new tests must never change how existing tests or the app behave.

1. **Root files in-app: only these, only added to.**
   - `pyproject.toml` — the test dependency group; a pytest section only when the repo has none (`references/scaffold-python.md` §3).
   - `uv.lock` (or the repo's lockfile) — updated by the dependency tool.
   - `package.json` — devDependencies and `test:*` scripts.
   - An existing task runner (Makefile, justfile, tox, nox) — new test targets.
   - `.gitignore` — `tests/.env`, `tests/.auth/`, `tests/reports/`.

   CI workflows come later, on demand (`references/ci-github-actions.md`). Leave `.python-version` and `requires-python` exactly as the team set them. Shared files are merged, never overwritten: add the smallest change in the format the repo already uses.
2. **Footprint preview.** Before writing anything, show the user every root file the scaffold will touch, marked **new** or **added to**, with the diff of each addition. Write after one approval.
3. **One pytest config.** If any of `pytest.ini`, `pyproject.toml [tool.pytest.ini_options]`, `setup.cfg [tool:pytest]`, or `tox.ini [pytest]` exists, extend that file. Create `[tool.pytest.ini_options]` only when none exists — `pytest.ini` silently wins over `pyproject.toml`, so two configs means one is ignored.
4. **Follow the repo's tooling.** Dependency tool (uv, poetry, pip-tools, plain pip), lint and type-check config, and the Python version stay as the team set them; add a test group in their format (e.g. `[tool.poetry.group.test.dependencies]`). New test code follows the team's linter; add no lint config of its own.
5. **Scope stricter options to new tests.** `-n auto`, `--timeout`, and `--reruns` go in the layer run commands and CI, never in global `addopts`. Add `--strict-markers` globally only after registering every marker the existing suite uses.
6. **Avoid fixture collisions.** In Assess, list existing fixtures (`pytest --fixtures`). Known clash: pytest-django provides `settings` — this skill's settings fixture is `env_settings`. Rename any other fixture that collides.
7. **Leave existing tests in place.** Do not move or rename them. Add `api/`, `e2e/`, `support/` beside them. If a root `tests/conftest.py` exists, append only the `pytest_plugins` line. When existing unit tests live flat in `tests/`, new unit tests follow that location; record it in `TESTING.md`.
8. **Env files stay inside `tests/`** (§3) so no framework loader (Next.js, Vite, Django, docker compose) picks them up.
9. **Existing test folders and frameworks.** When tests live in `test/`, `__tests__/`, `e2e/`, or similar, add new layers inside that folder and record it in `TESTING.md`; read `tests/` in this skill as that folder. When a layer already uses another framework (Cypress for e2e, Jest for unit), stop and ask the user before adding a second one beside it. If the user keeps the existing framework, write that layer's tests in it, in its existing style.

## 3. Env file convention

- **Every test variable starts with `TEST_`** — `TEST_TARGET_ENV`, `TEST_BASE_URL`, `TEST_API_URL`, `TEST_DATABASE_URL`, `TEST_ROLES`, `TEST_ROLE_<NAME>_USERNAME` / `_PASSWORD`. Nothing collides with the app's own variables (Vite's built-in `BASE_URL`, an app `BASE_URL`, `DATABASE_URL`).
- **Files live in `ENV_DIR`** — `tests/` in-app, the repo root in a standalone repo.
- **Precedence:** exported variables (shell, CI) → `.env` (secrets, personal overrides) → `.env.<TEST_TARGET_ENV>` (committed, non-secret). `TEST_TARGET_ENV` may itself be set in `.env`, defaulting to `local`.
- **Secrets** live only in the git-ignored `.env` or in CI secrets. Committed files hold URLs and non-secret values.
- **Production** has no env file. A production run requires exporting its values deliberately; CI supplies them from secrets behind a protected environment (`references/ci-github-actions.md` §5).

## 4. Runners

| Stack | Unit | API | UI |
|-------|------|-----|----|
| Python | pytest | pytest + httpx | pytest + pytest-playwright (sync API) |
| TS | Vitest | Playwright Test (`request`) | Playwright Test |

When the repo already uses another runner for a layer, keep it.

## 5. Tagging

| Tag | Python | TS (Playwright) | Meaning |
|-----|--------|-----------------|---------|
| smoke | `@pytest.mark.smoke` | `@smoke` in title | P0, gates merge, safe on production if read-only |
| regression | `@pytest.mark.regression` | `@regression` | full set |
| writes | `@pytest.mark.writes` | `@writes` | mutates data |
| slow | `@pytest.mark.slow` | `@slow` | opt-in |

## 6. Reporting

- JUnit XML from every runner into `tests/reports/` (standalone: `reports/`) — CI reads it once CI is set up.
- Human report: pytest-html (`--html=tests/reports/report.html --self-contained-html`), Playwright HTML, Locust `--html`.
- On e2e failure: trace, screenshot, video retained.

## 7. Proof the scaffold works

The scaffold is done only when it ships **one working example test per layer in scope**, plus U-00, and they pass Verify — including the prove-fail check:

- **U-00** — `.env.example` documents every setting (§3 and the stack file's settings). Runs with no `.env`.
- **I-00** — a health endpoint or known-stable GET returns 200 and matches its schema.
- **E-00** — the home (or landing) page loads anonymously and a key element is visible.

If Assess found no health endpoint or stable page, ask the user for one.

The example tests for each stack are in its stack file.

## 8. Output of this phase

The scaffold files, the approved footprint preview (root files and diffs) and dependency list, the type-checking answer, U-00 / I-00 / E-00 passing Verify, and the exact run command per layer recorded in `tests/TESTING.md`.
