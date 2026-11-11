# Verification

Loaded at `[SDET: Verify]`. Every task ends here. A test counts as done only when it has run green and — for key tests — been seen failing for the right reason.

## 1. Run green

1. Make sure the system-under-test is available: start it with the project's run command, or tell the user the exact command and wait for confirmation that it is up.
2. Run only the tests written or changed in this task first, then the layer they belong to, so regressions in neighbors show up.
3. All must pass. When a test fails:
   - **Test is wrong** (bad selector, wrong expected value from a misread spec) → fix the test and rerun.
   - **Product looks wrong** → stop and report it to the user as a finding: test ID, expected, actual, evidence (response body, trace, screenshot). Keep the assertion on correct behavior; mark the test `xfail` / `test.fail()` only if the user agrees, with a reference to the bug.
   - **Environment** (service down, missing env var) → fix the setup or tell the user what is missing.
4. Run new tests a second time. A test that passes once and fails once is not done — find the source of non-determinism (shared data, timing, order) and fix it.

## 2. Prove-fail check

Required for every P0 test and for the first test of each new assertion type (first schema check, first network assertion, first threshold). Optional for the rest.

For each test checked:

1. **Break the asserted condition** in the smallest way that should make it fail:
   - change the expected value in the assertion (`200` → `201`, expected text → a different string), or
   - feed the input that violates the rule (valid payload → payload missing a required field, but keep the happy-path assertion), or
   - for schema checks, remove a required field from the schema's copy of the response or tighten the schema type, or
   - for perf thresholds, set the threshold below the observed value.
2. Run the test and confirm it **fails, and fails on the assertion you broke** — not on an import error, timeout, or setup failure.
3. Restore the original and confirm it passes again.
4. Leave no mutation in the code. Check the diff afterwards.

Mutate the test side, not the product code, unless the user asks for product-side mutation. If the test still passes after the break, it is not asserting what it claims — rewrite it and repeat.

## 3. Report

End the phase with a short report:

```
[SDET: Verify]

Added / changed: 12 tests (U-01..U-06, I-10..I-13, E-01, E-02)
Run: 12 passed, 0 failed (2 consecutive runs)
Prove-fail: U-01, I-10, I-11, E-01 — each failed on the broken assertion, green after restore
Not verified: E-02 — needs staging credentials (TEST_ROLE_MEMBER_USERNAME missing for staging)
Findings: I-12 — PATCH /orders/{id} returns 500 for qty=0; expected 422. Evidence: response body below.
Run commands:
  uv run pytest tests/unit tests/api -n auto
  uv run pytest tests/e2e -m smoke -n auto
```

State any step that was skipped and why. A test that could not be run is reported as unverified, not as done.
