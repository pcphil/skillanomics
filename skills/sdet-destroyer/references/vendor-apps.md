# Vendor apps (black-box mode)

Loaded at `[SDET: Assess]` when the system-under-test is a vendor product: hosted (SaaS) or self-hosted, with UI and/or API the team configures but does not own the source of. The live app and the vendor's API spec replace the source code as the source of truth; nothing can be added to the app (no `data-testid`, no new endpoints).

## 1. Detection

Enter vendor mode when either holds:

- Assess finds no application source in the repo (only config, integration code, or nothing), and the target is a running app.
- The user names a vendor product or says the app is bought, not built.

State it in the Assess summary so a wrong detection gets corrected: `Vendor mode: <product>, <hosted | self-hosted>`. Ask which if unknown.

Consequences:

- The suite lives in a **standalone test repo** (`references/architecture.md` §1). Python by default; confirm with the user.
- Assess reads the live app (`references/browser-exploration.md`), the vendor spec, the org's integration code if any, and vendor release notes the user points to.

## 2. What to test

Test what the org owns, not the vendor's core product:

| In scope | Out of scope |
|----------|--------------|
| Configuration: custom fields, layouts, picklists, validation rules | Vendor's built-in features working as documented |
| Custom workflows, automations, approval chains | Vendor platform uptime and performance |
| Roles, permissions, sharing rules — who can see and do what | Vendor UI components themselves |
| Integrations: inbound and outbound APIs, webhooks, data sync, field mappings | |
| Data migrations and imports | |

Add to the strategy's risk list:

- **Vendor releases** — every vendor upgrade can change UI markup, API shapes, and behavior of configuration. Rate High when the vendor upgrades on its own schedule. Cover it with a full regression run in the sandbox before and after each upgrade window (`references/ci-github-actions.md` §4, manual dispatch).
- **Configuration drift** — admins change config in production directly; tests catch changes that break agreed behavior.

## 3. Locators

Vendor markup is generated and cannot be changed. Choose locators in this order:

1. Role + accessible name.
2. Label, placeholder, visible text.
3. A stable attribute the vendor renders (`name`, `aria-*`, a documented `data-*`) — only after it passes the stability check.

Reject generated values: numeric or hashed IDs and classes, for example `__button12` or `__xmlview0--` (SAP UI5), `j_id0:j_id12`, React `:r3:`, CSS-in-JS `css-1x2y3z`, GUIDs, and anything with a counter in it.

**Stability check** (with a browser-control tool):

1. Session A: open the screen, take an accessibility snapshot, record each candidate locator and the attributes of elements without an accessible name.
2. Close the context, start a fresh session (cleared storage, new login), reach the same screen state, and record again.
3. Keep only locators and attributes identical in both. Anything that changed is generated — drop it.

Structure:

- **Iframes** — locate through the frame: `page.frame_locator("iframe[name=gsft_main]")` (ServiceNow classic UI is an example). Note the frame in the page object.
- **Shadow DOM** — Playwright locators pierce open shadow roots by default (Salesforce Lightning and ServiceNow Next Experience use them); XPath does not, so do not use XPath.
- **No stable locator at all** — record the element under "Fragile locators" in the Vendor section of `tests/TESTING.md`, use the least-bad locator, and mark it in the page object:

```python
# FRAGILE (vendor markup, no accessible name): re-check after each vendor upgrade
self.save = page.locator("[data-region-name='footer'] button").last
```

Without a browser tool, use Playwright codegen against the app (`references/browser-exploration.md` §5) and apply the same rejection rules.

## 4. Login: SSO and MFA

- **Exploration:** the user logs in by hand, as in `references/browser-exploration.md` §2.
- **Test runs:** ask the vendor admin for a dedicated integration user, either exempt from MFA or with a TOTP secret. Store the secret as `TEST_ROLE_<NAME>_TOTP_SECRET` (`.env` / CI secrets; add a commented placeholder to `.env.example`). Never bypass MFA or disable it for a real user.
- **Fallback:** saved storage state the user refreshes by hand. Record its expiry in `TESTING.md`; nightly runs fail when it lapses.

Vendor apps often have no API login that yields a browser session. In vendor mode, storage state may be created by logging in through the UI once per role:

```python
# tests/support/fixtures/auth.py (vendor mode) — locators come from exploration
import pyotp

def _ui_login(browser, env, role: str, path: str) -> None:
    creds = env.role(role)
    ctx = browser.new_context(base_url=env.require("base_url"))
    page = ctx.new_page()
    page.goto("/")
    page.get_by_label("Username").fill(creds.username)
    page.get_by_label("Password").fill(creds.password)
    page.get_by_role("button", name="Log in").click()
    if creds.totp_secret:
        page.get_by_label("Verification code").fill(pyotp.TOTP(creds.totp_secret).now())
        page.get_by_role("button", name="Verify").click()
    page.wait_for_url("**/home**")
    ctx.storage_state(path=path)
    ctx.close()
```

```ts
// tests/e2e/auth.setup.ts (vendor mode, excerpt)
import * as OTPAuth from "otpauth";
const code = new OTPAuth.TOTP({ secret: OTPAuth.Secret.fromBase32(creds.totpSecret) }).generate();
```

An identity provider usually rejects a TOTP code used twice, and some vendors limit concurrent sessions per user. Log in **once per role per run**, not once per worker: TS does this in the setup project; Python writes `tests/.auth/<role>.json` in a CI step before pytest (`uv run python -m tests.support.login`), and `storage_state_for` uses that file when it exists.

## 5. Vendor API spec (Swagger / OpenAPI)

The user supplies a file path or URL (common URLs: `/swagger.json`, `/v3/api-docs`, `/openapi.json`, or a download from the vendor's developer portal). If the URL needs a login, ask the user to download the file.

Files in `tests/support/models/`:

| File | Committed | Purpose |
|------|-----------|---------|
| `refresh.py` / `refresh.ts` | yes | fetches the spec, converts Swagger 2.0 → OpenAPI 3, writes `openapi.json` |
| `openapi.json` | yes | the vendor spec as fetched (3.x), never hand-edited |
| `openapi.overrides.json` | yes | corrections where the spec is wrong about reality |
| `generate.py` / `generate.ts` | yes | merges overrides, writes `openapi.merged.json` and the models |
| `openapi.merged.json`, `generated.py` / `api.d.ts` | yes | generated; never hand-edited |

Run `refresh` by hand when the vendor version changes, then `generate`. Record source, vendor version, and date in the Vendor section of `TESTING.md`.

```python
# tests/support/models/refresh.py — usage: uv run python -m tests.support.models.refresh <path-or-url>
import json
import shutil
import subprocess
import sys
import tempfile
from pathlib import Path

import httpx
import yaml

HERE = Path(__file__).parent

def main(source: str) -> None:
    if source.startswith(("http://", "https://")):
        resp = httpx.get(source, timeout=30, follow_redirects=True)
        resp.raise_for_status()
        text = resp.text
    else:
        text = Path(source).read_text(encoding="utf-8")
    spec = yaml.safe_load(text)  # reads JSON and YAML
    if str(spec.get("swagger", "")).startswith("2."):
        npx = shutil.which("npx")
        if not npx:
            raise SystemExit("Swagger 2.0 needs Node for conversion (npx swagger2openapi)")
        with tempfile.TemporaryDirectory() as tmp:
            src, out = Path(tmp) / "swagger.json", Path(tmp) / "openapi.json"
            src.write_text(json.dumps(spec), encoding="utf-8")
            subprocess.run([npx, "--yes", "swagger2openapi@7", str(src), "-o", str(out)], check=True)
            spec = json.loads(out.read_text(encoding="utf-8"))
    (HERE / "openapi.json").write_text(json.dumps(spec, indent=2, sort_keys=True) + "\n", encoding="utf-8")
    print(f"openapi.json written: {spec['info'].get('title')} {spec['info'].get('version')}")

if __name__ == "__main__":
    main(sys.argv[1])
```

```python
# tests/support/models/generate.py — usage: uv run python -m tests.support.models.generate
import json
import subprocess
from pathlib import Path

HERE = Path(__file__).parent

def merge(base: dict, patch: dict) -> dict:
    """Objects merge key by key; lists and scalars in the patch replace."""
    for key, value in patch.items():
        if isinstance(value, dict) and isinstance(base.get(key), dict):
            merge(base[key], value)
        else:
            base[key] = value
    return base

spec = json.loads((HERE / "openapi.json").read_text(encoding="utf-8"))
overrides = HERE / "openapi.overrides.json"
if overrides.exists():
    merge(spec, json.loads(overrides.read_text(encoding="utf-8")))
merged = HERE / "openapi.merged.json"
merged.write_text(json.dumps(spec, indent=2, sort_keys=True) + "\n", encoding="utf-8")
subprocess.run(
    ["datamodel-codegen", "--input", str(merged), "--input-file-type", "openapi",
     "--output-model-type", "pydantic_v2.BaseModel", "--extra-fields", "allow",
     "--output", str(HERE / "generated.py")],
    check=True,
)
```

TS: `refresh.ts` does the same with `npx swagger2openapi`; `generate.ts` applies the same merge, writes `openapi.merged.json`, then runs `openapi-typescript openapi.merged.json -o api.d.ts`. ajv validates against `openapi.merged.json` component schemas.

An override is a partial spec. Example — the spec says `total` is a number, the API returns a string:

```json
{ "components": { "schemas": { "Order": { "properties": { "total": { "type": "string" } } } } } }
```

Record the reason for each override in the Vendor section of `TESTING.md` (JSON has no comments).

## 6. Schema strictness and drift

- **Extra fields are allowed** on vendor responses (`--extra-fields allow`; no `additionalProperties: false` added). A field the vendor adds is not a failure for the org. This replaces the closed-shape rule in `references/api.md` §2.
- **Required fields and types stay strict.** A missing required field or a changed type fails the test.

**Drift check** — run during exploration, after `refresh`, and whenever a schema assertion fails:

1. Take real responses: browser network capture (`references/browser-exploration.md` §4.3), API test failures, or GETs on stable records with a test account.
2. Compare each against the merged spec: missing required fields, type mismatches, undocumented enum values, undocumented status codes, endpoints missing from the spec. Note extra fields as informational.
3. Report to the user as findings:

```
Spec drift (vendor v2026.3, staging):
| Endpoint | Field | Spec | Actual | Suggested |
|----------|-------|------|--------|-----------|
| GET /orders/{id} | total | number | "12.50" (string) | override |
| GET /orders | status | enum [open, closed] | "on_hold" | override, or vendor bug report |
```

4. The user decides per row: the spec is wrong → add an override; the API is wrong → report to the vendor and keep the test asserting the spec.

Raw captured responses stay out of the repo (`references/browser-exploration.md` §4.3, step 5).

## 7. Which endpoints get tests

Generate models for the whole spec, but write tests only for endpoints the org uses: those its journeys call (network capture), those its integration code calls, and those the user names. List them in the Endpoints table of `TESTING.md` with their source. An endpoint not in that list gets no test until someone uses it.

## 8. Load testing

- **Vendor-hosted (SaaS):** off by default — vendor terms usually forbid load testing and can throttle or suspend the tenant. Allowed only against a sandbox, and only after the user confirms the vendor's written approval. Record the approval (who, when, limits) in `TESTING.md` and stay within its limits.
- **Self-hosted vendor product:** treat as the org's own app — staging only (`references/performance.md`).

## 9. Test data in shared sandboxes

Vendor sandboxes are shared across teams, and aborted runs leave records behind.

- Every record a test creates carries a run prefix in a visible name field:

```python
# tests/factories/naming.py
import os
from uuid import uuid4

RUN_ID = os.environ.get("TEST_RUN_ID") or uuid4().hex[:6]  # CI sets TEST_RUN_ID so all workers share it

def tagged(label: str) -> str:
    return f"sdet-{RUN_ID}-{label}-{uuid4().hex[:6]}"
```

- Teardown deletes what the test created, as everywhere (`references/test-data.md`).
- A sweep script deletes prefixed records older than 24 hours; CI runs it nightly (`references/ci-github-actions.md` §4):

```python
# tests/support/sweep.py — usage: uv run python -m tests.support.sweep [--dry-run]
import sys
from datetime import UTC, datetime, timedelta

import httpx

from tests.settings import get_settings
from tests.support.clients import Api
from tests.support.fixtures.api import _token

PREFIX = "sdet-"
SWEEPABLE = ("accounts", "contacts")  # Api attributes for the resources tests create

def main(dry_run: bool) -> None:
    env = get_settings()
    if env.target_env == "production":
        raise SystemExit("sweep runs only against local or sandbox environments")
    cutoff = datetime.now(UTC) - timedelta(hours=24)
    headers = {"Authorization": f"Bearer {_token(env, 'admin')}"}
    with httpx.Client(base_url=env.api_base, headers=headers, timeout=30) as http:
        api = Api(http)
        for name in SWEEPABLE:
            client = getattr(api, name)
            for record in client.list_prefixed(PREFIX):  # vendor search: name starts with PREFIX
                if record.created_at < cutoff:
                    print(f"{'would delete' if dry_run else 'deleting'} {name}/{record.id} {record.name}")
                    if not dry_run:
                        client.delete(record.id)

if __name__ == "__main__":
    main("--dry-run" in sys.argv)
```

Run it with `--dry-run` first and show the user the list before the first real sweep.

## 10. TESTING.md: Vendor section

Add to `tests/TESTING.md` (template in `references/strategy.md`):

```markdown
## Vendor
- Product: <name>, <hosted | self-hosted>, version <x> (checked <date>)
- Upgrade windows: <dates or cadence>; regression runs before and after
- Spec: <source URL or file>, vendor version <x>, fetched <date>
- Overrides: <field → reason>
- Fragile locators: <page object → element → why>
- Login: <integration user per role, MFA method, storage-state expiry if used>
- Load testing: <not approved | approved by <who> on <date>, limits>
```
