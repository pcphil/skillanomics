# Unit and integration tests

Loaded at `[SDET: Unit]`. Unit tests prove pure logic fast and exhaustively; integration tests prove that code works with its real collaborators (DB, cache, framework) at one boundary. Together they carry most of the suite.

## 1. What goes where

| Unit | Integration |
|------|-------------|
| validators, parsers, calculations, state machines, formatters | repository + real DB, route + framework test client, service + queue |
| no I/O, no network, no clock | real dependency in a container, or the framework's in-process client |
| milliseconds | under a few seconds each |

Mock only at the **edge you do not own** (third-party HTTP, email, payment provider). Mocking your own modules couples the test to the implementation and hides integration bugs.

## 2. Case row → test

One row → one test, case ID in the name. A row listing several inputs → one parametrized test.

### pytest

```python
import pytest
from app.pricing import apply_discount

# U-01 discount applies percentage to subtotal
def test_u01_discount_applies_percentage():
    assert apply_discount(subtotal=100_00, percent=15) == 85_00

# U-02 discount rejects out-of-range percent
@pytest.mark.parametrize("percent", [-1, 101, 1000])
def test_u02_discount_rejects_out_of_range(percent):
    with pytest.raises(ValueError, match="percent"):
        apply_discount(subtotal=100_00, percent=percent)
```

### Vitest

```ts
import { describe, expect, it } from "vitest";
import { applyDiscount } from "../../src/pricing";

describe("pricing", () => {
  it("U-01 discount applies percentage to subtotal", () => {
    expect(applyDiscount({ subtotal: 100_00, percent: 15 })).toBe(85_00);
  });

  it.each([-1, 101, 1000])("U-02 discount rejects out-of-range percent %i", (percent) => {
    expect(() => applyDiscount({ subtotal: 100_00, percent })).toThrow(/percent/);
  });
});
```

## 3. Integration patterns

### Framework in-process client (no server needed)

Prefer the framework's own test client when it exists — it exercises routing, middleware, validation, and serialization without a network hop.

```python
# FastAPI example; Flask/Django have equivalents
from fastapi.testclient import TestClient
from app.main import app

client = TestClient(app)

# I-01 create order returns 201 with id
def test_i01_create_order(db_session, user_factory):
    user = user_factory()
    resp = client.post("/orders", json={"user_id": user.id, "items": [{"sku": "A1", "qty": 2}]})
    assert resp.status_code == 201
    assert resp.json()["id"]
```

```ts
// Express/Fastify with supertest or app.inject; Next.js route handlers can be called directly
import request from "supertest";
import { app } from "../../src/app";

it("I-01 create order returns 201 with id", async () => {
  const res = await request(app).post("/orders").send({ userId: "u1", items: [{ sku: "A1", qty: 2 }] });
  expect(res.status).toBe(201);
  expect(res.body.id).toBeTruthy();
});
```

### Real database

Run the real engine in a container (see `references/test-data.md`), not SQLite standing in for Postgres — dialect differences hide real bugs. Wrap each test in a transaction that rolls back, or truncate touched tables in teardown.

### Mocking the edge

```python
# third-party payment client is the edge; patch where it is looked up
def test_i04_payment_decline_marks_order_failed(mocker, order_factory):
    mocker.patch("app.orders.payments.charge", side_effect=PaymentDeclined("card_declined"))
    order = order_factory(status="pending")
    process_order(order.id)
    assert get_order(order.id).status == "failed"
```

```ts
import { vi } from "vitest";
import * as payments from "../../src/payments";

it("I-04 payment decline marks order failed", async () => {
  vi.spyOn(payments, "charge").mockRejectedValue(new PaymentDeclined("card_declined"));
  const order = await makeOrder({ status: "pending" });
  await processOrder(order.id);
  expect((await getOrder(order.id)).status).toBe("failed");
});
```

## 4. Determinism

- Freeze time where behavior depends on it: `freezegun` / `time-machine` (Python), `vi.useFakeTimers()` + `vi.setSystemTime()` (Vitest).
- Seed randomness: `Faker.seed(...)`, `faker.seed(...)`.
- Never depend on test order or another test's data.

## 5. Coverage discipline

- Every requirement: happy path + at least one negative/edge.
- Boundaries explicitly: zero, one, max, max+1, empty, null, unicode, very long.
- Every test ends on an assertion of the expected result — a test that only "runs without error" proves nothing.
- Read the tests back against the strategy's risk areas before moving to Verify.
