# API tests

Loaded at `[SDET: API]`. API tests hit a running service over HTTP and prove its contract: status codes, response shape, error shape, auth, and side effects. They run against local or staging; against production only the read-only smoke subset runs, after explicit user confirmation.

## 1. What every endpoint gets

For each endpoint in scope, cover the rows that apply:

| Check | Example expected result |
|-------|-------------------------|
| Happy path | 200/201 + body matches schema |
| Validation | 400/422 + error body names the bad field |
| Auth missing | 401 |
| Auth wrong role / other tenant | 403 or 404 (whichever the API promises — and consistently) |
| Not found | 404 + error schema |
| Conflict / idempotency | 409, or repeated request returns the same result |
| Side effect | follow-up GET shows the created/updated state |
| Pagination / filtering | page size respected, `next` link correct, filters exclude correctly |

## 2. Schema validation

Validate every success and error response against a schema, in addition to asserting key values. Schema checks catch added, removed, renamed, and re-typed fields that value assertions miss.

Source of schemas, in order of preference:

1. The service's OpenAPI spec — extract component schemas into `tests/api/schemas/`.
2. Existing typed models in the codebase (pydantic models, zod schemas) — import them.
3. Hand-written JSON Schema, kept next to the tests.

Set `additionalProperties: false` on response schemas where the API promises a closed shape, so unexpected fields fail the test.

### Python (httpx + jsonschema)

```python
import json
import pathlib
import pytest
from jsonschema import validate

SCHEMAS = pathlib.Path(__file__).parent / "schemas"

def schema(name):
    return json.loads((SCHEMAS / f"{name}.json").read_text())

# I-10 get order returns order schema
@pytest.mark.api
@pytest.mark.smoke
def test_i10_get_order(api, existing_order_id):
    resp = api.get(f"/orders/{existing_order_id}")
    assert resp.status_code == 200
    validate(resp.json(), schema("order"))
    assert resp.json()["id"] == existing_order_id

# I-11 create order rejects empty items
@pytest.mark.api
@pytest.mark.writes
def test_i11_create_order_rejects_empty_items(api):
    resp = api.post("/orders", json={"items": []})
    assert resp.status_code == 422
    validate(resp.json(), schema("error"))
    assert "items" in resp.text
```

With pydantic models available, `Order.model_validate(resp.json())` replaces `validate(...)`.

### TS (Playwright `request` + ajv)

```ts
import { test, expect } from "@playwright/test";
import Ajv from "ajv";
import addFormats from "ajv-formats";
import orderSchema from "./schemas/order.json";
import errorSchema from "./schemas/error.json";

const ajv = addFormats(new Ajv({ allErrors: true }));
const validOrder = ajv.compile(orderSchema);
const validError = ajv.compile(errorSchema);

test("I-10 get order returns order schema @api @smoke", async ({ request }) => {
  const res = await request.get(`/orders/${process.env.EXISTING_ORDER_ID}`);
  expect(res.status()).toBe(200);
  const body = await res.json();
  expect(validOrder(body), JSON.stringify(validOrder.errors)).toBe(true);
});

test("I-11 create order rejects empty items @api @writes", async ({ request }) => {
  const res = await request.post("/orders", { data: { items: [] } });
  expect(res.status()).toBe(422);
  const body = await res.json();
  expect(validError(body), JSON.stringify(validError.errors)).toBe(true);
});
```

With zod schemas in the codebase, `OrderSchema.parse(body)` replaces ajv.

Auth for the `request` fixture: set `extraHTTPHeaders` in `playwright.config.ts` from an env token, or create an authenticated `request.newContext()` in a fixture.

## 3. Side effects and cleanup

- Assert side effects through the API itself (follow-up GET), or through the DB in integration tests — not by trusting the POST's response alone.
- Every test that creates a resource deletes it in teardown, or creates it under a test-scoped tenant/user that is removed after.
- Use unique identifiers per test (`f"test-{uuid4()}"`) so parallel runs never collide.

## 4. Auth tests

- Obtain tokens through the real auth flow once per session; cache them in a fixture.
- Keep separate fixtures per role (`admin_api`, `user_api`, `anon_api`) so access-control rows read clearly.
- For multi-tenant APIs, always include one cross-tenant read and one cross-tenant write that must be refused.

## 5. Timeouts and resilience

- Set an explicit client timeout (e.g. 10 s) so a hung service fails the test instead of the job.
- Assert latency only in the performance phase; functional API tests assert correctness.
