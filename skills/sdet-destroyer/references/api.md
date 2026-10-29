# API tests

Loaded at `[SDET: API]`. API tests hit a running service over HTTP and prove its contract: status codes, response shape, error shape, auth, and side effects. They run against local or staging; against production only the read-only smoke subset runs, after explicit user confirmation.

Stack: Python uses pytest + httpx. TS uses Playwright Test's `request`. Both go through a client layer.

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

## 2. Schemas and models

Validate every success and error response against a schema, in addition to asserting key values. Schema checks catch added, removed, renamed, and re-typed fields that value assertions miss.

Source, decided once in Scaffold and recorded in `tests/TESTING.md`:

1. **OpenAPI spec exists → generate.** Commit a generate script in `tests/support/models/` and the generated output. Never hand-edit generated files; rerun the script when the spec changes.
   - Python: `datamodel-codegen --input openapi.yaml --input-file-type openapi --output-model-type pydantic_v2.BaseModel --extra-fields forbid --output tests/support/models/generated.py`
   - TS: bundle the spec to `tests/support/models/openapi.json` and generate types with `openapi-typescript openapi.json -o tests/support/models/api.d.ts`. Validate with ajv against the spec's component schemas (OpenAPI 3.1 is JSON Schema 2020-12; for a 3.0 spec, convert schemas in the generate script so `nullable` is honored).
2. **No spec → hand-write.** Python: pydantic models in `tests/support/models/`. TS: JSON Schema files in `tests/support/models/schemas/` plus hand-written types. Draft them from codebase types (route handlers, serializers, existing pydantic/zod models) and, when a browser-control tool is available, from captured responses (`references/browser-exploration.md`).

Closed shapes: forbid extra fields (`--extra-fields forbid`, `additionalProperties: false`) where the API promises them, so unexpected fields fail the test.

## 3. Client layer

One client per resource in `tests/support/clients/`. Two methods per operation:

- `create(payload)` → raw response. API tests assert on it directly.
- `create_ok(payload)` / `createOk(payload)` → asserts 2xx, validates the schema, returns the typed model. Used for setup — in API tests and in UI tests alike.

Clients hold paths, headers, and serialization. Tests hold assertions.

### Python

```python
# tests/support/clients/base.py
from typing import TypeVar
import httpx
from pydantic import BaseModel

M = TypeVar("M", bound=BaseModel)

def expect_ok(resp: httpx.Response, model: type[M]) -> M:
    if not resp.is_success:
        raise AssertionError(f"{resp.request.method} {resp.request.url} -> {resp.status_code}: {resp.text[:500]}")
    return model.model_validate(resp.json())
```

```python
# tests/support/clients/orders.py
import httpx
from tests.support.clients.base import expect_ok
from tests.support.models import Order

class OrdersClient:
    def __init__(self, http: httpx.Client):
        self.http = http

    def create(self, payload: dict) -> httpx.Response:
        return self.http.post("/orders", json=payload)

    def create_ok(self, payload: dict) -> Order:
        return expect_ok(self.create(payload), Order)

    def get(self, order_id: str) -> httpx.Response:
        return self.http.get(f"/orders/{order_id}")

    def get_ok(self, order_id: str) -> Order:
        return expect_ok(self.get(order_id), Order)

    def delete(self, order_id: str) -> httpx.Response:
        return self.http.delete(f"/orders/{order_id}")
```

```python
# tests/support/clients/__init__.py
from tests.support.clients.health import HealthClient
from tests.support.clients.orders import OrdersClient

class Api:
    def __init__(self, http):
        self.http = http
        self.health = HealthClient(http)
        self.orders = OrdersClient(http)
```

### TS

```ts
// tests/support/models/index.ts
import Ajv from "ajv/dist/2020";
import addFormats from "ajv-formats";
import spec from "./openapi.json";
import type { components } from "./api";

const ajv = addFormats(new Ajv({ allErrors: true, strict: false }));
ajv.addSchema(spec, "spec");
const schema = <T>(name: string) => ajv.getSchema<T>(`spec#/components/schemas/${name}`)!;

export type Order = components["schemas"]["Order"];
export type OrderInput = components["schemas"]["OrderInput"];
export type ApiError = components["schemas"]["Error"];
export const validateOrder = schema<Order>("Order");
export const validateError = schema<ApiError>("Error");
```

```ts
// tests/support/clients/base.ts
import type { APIResponse } from "@playwright/test";
import type { ValidateFunction } from "ajv";

export async function expectOk<T>(res: APIResponse, validate: ValidateFunction<T>): Promise<T> {
  if (!res.ok()) throw new Error(`${res.url()} -> ${res.status()}: ${(await res.text()).slice(0, 500)}`);
  const body: unknown = await res.json();
  if (!validate(body)) throw new Error(`schema mismatch at ${res.url()}: ${JSON.stringify(validate.errors)}`);
  return body;
}
```

```ts
// tests/support/clients/orders.ts
import type { APIRequestContext } from "@playwright/test";
import { expectOk } from "./base";
import { validateOrder, type Order, type OrderInput } from "../models";

export class OrdersClient {
  constructor(private readonly request: APIRequestContext) {}

  create(payload: OrderInput) {
    return this.request.post("/orders", { data: payload });
  }

  async createOk(payload: OrderInput): Promise<Order> {
    return expectOk(await this.create(payload), validateOrder);
  }

  get(id: string) {
    return this.request.get(`/orders/${id}`);
  }

  async getOk(id: string): Promise<Order> {
    return expectOk(await this.get(id), validateOrder);
  }

  delete(id: string) {
    return this.request.delete(`/orders/${id}`);
  }
}
```

```ts
// tests/support/clients/index.ts
import type { APIRequestContext } from "@playwright/test";
import { HealthClient } from "./health";
import { OrdersClient } from "./orders";

export class Api {
  readonly health: HealthClient;
  readonly orders: OrdersClient;

  constructor(readonly request: APIRequestContext) {
    this.health = new HealthClient(request);
    this.orders = new OrdersClient(request);
  }
}
```

## 4. Auth fixtures per role

Each role in `TEST_ROLES` gets an authenticated `Api`, logged in through the API once per session (Python; once per xdist worker) or worker (TS), plus `anon` with no credentials. The login path and token field below are placeholders — use what Assess found (bearer token, session cookie, OAuth client credentials).

### Python

```python
# tests/support/fixtures/api.py — registered via pytest_plugins in tests/conftest.py
import httpx
import pytest
from tests.settings import Settings
from tests.support.clients import Api

def _token(env: Settings, role: str) -> str:
    creds = env.role(role)
    resp = httpx.post(f"{env.api_base}/auth/login",
                      json={"username": creds.username, "password": creds.password}, timeout=10)
    if not resp.is_success:
        raise RuntimeError(f"login failed for role {role}: {resp.status_code}")
    return resp.json()["token"]

@pytest.fixture(scope="session")
def api_as(env_settings):
    clients: dict[str | None, httpx.Client] = {}

    def get(role: str | None) -> Api:
        if role not in clients:
            headers = {"Authorization": f"Bearer {_token(env_settings, role)}"} if role else {}
            clients[role] = httpx.Client(base_url=env_settings.api_base, headers=headers, timeout=10)
        return Api(clients[role])

    yield get
    for client in clients.values():
        client.close()

@pytest.fixture(scope="session")
def anon_api(api_as):
    return api_as(None)

@pytest.fixture(scope="session")
def member_api(api_as):
    return api_as("member")

@pytest.fixture(scope="session")
def admin_api(api_as):
    return api_as("admin")
```

Add one `<role>_api` fixture per role the strategy names.

### TS

```ts
// tests/api/fixtures.ts
import { test as base, request as pwRequest, type APIRequestContext } from "@playwright/test";
import { settings, roles } from "../settings";
import { Api } from "../support/clients";

async function contextFor(role: string | null): Promise<APIRequestContext> {
  if (!role) return pwRequest.newContext({ baseURL: settings.apiBase });
  const login = await pwRequest.newContext({ baseURL: settings.apiBase });
  const res = await login.post("/auth/login", { data: roles[role] });
  if (!res.ok()) throw new Error(`login failed for role ${role}: ${res.status()}`);
  const { token } = await res.json();
  await login.dispose();
  return pwRequest.newContext({ baseURL: settings.apiBase, extraHTTPHeaders: { Authorization: `Bearer ${token}` } });
}

type ApiFixtures = { anonApi: Api; memberApi: Api; adminApi: Api };

function apiFixture(role: string | null) {
  return [
    async ({}, use: (api: Api) => Promise<void>) => {
      const ctx = await contextFor(role);
      await use(new Api(ctx));
      await ctx.dispose();
    },
    { scope: "worker" },
  ] as const;
}

export const test = base.extend<{}, ApiFixtures>({
  anonApi: apiFixture(null),
  memberApi: apiFixture("member"),
  adminApi: apiFixture("admin"),
});
export { expect } from "@playwright/test";
```

UI fixtures extend this `test`, so browser tests get the same clients for setup.

## 5. Tests

### Python

```python
# tests/api/test_orders.py
import pytest
from tests.factories.order import order_payload
from tests.support.models import ApiError, Order

@pytest.fixture
def order(member_api):
    created = member_api.orders.create_ok(order_payload())
    yield created
    member_api.orders.delete(created.id)

# I-10 get order returns order schema
@pytest.mark.api
@pytest.mark.writes
def test_i10_get_order(member_api, order):
    resp = member_api.orders.get(order.id)
    assert resp.status_code == 200
    assert Order.model_validate(resp.json()).id == order.id

# I-11 create order rejects empty items
@pytest.mark.api
@pytest.mark.writes
def test_i11_create_order_rejects_empty_items(member_api):
    resp = member_api.orders.create(order_payload(items=[]))
    assert resp.status_code == 422
    ApiError.model_validate(resp.json())
    assert "items" in resp.text

# I-12 other member cannot read the order
# needs a second role, e.g. TEST_ROLES=member,other_member,admin, with an other_member_api fixture
@pytest.mark.api
@pytest.mark.writes
def test_i12_other_member_cannot_read_order(other_member_api, order):
    assert other_member_api.orders.get(order.id).status_code == 404

# I-13 anonymous request is rejected
@pytest.mark.api
@pytest.mark.smoke
def test_i13_anonymous_rejected(anon_api):
    assert anon_api.orders.get("any-id").status_code == 401
```

### TS

```ts
// tests/api/orders.spec.ts
import { test, expect } from "./fixtures";
import { orderPayload } from "../factories/order";
import { validateError, validateOrder } from "../support/models";

test.describe("orders", () => {
  test("I-10 get order returns order schema @writes", async ({ memberApi }) => {
    const order = await memberApi.orders.createOk(orderPayload());
    try {
      const res = await memberApi.orders.get(order.id);
      expect(res.status()).toBe(200);
      const body = await res.json();
      expect(validateOrder(body), JSON.stringify(validateOrder.errors)).toBe(true);
      expect(body.id).toBe(order.id);
    } finally {
      await memberApi.orders.delete(order.id);
    }
  });

  test("I-11 create order rejects empty items @writes", async ({ memberApi }) => {
    const res = await memberApi.orders.create(orderPayload({ items: [] }));
    expect(res.status()).toBe(422);
    const body = await res.json();
    expect(validateError(body), JSON.stringify(validateError.errors)).toBe(true);
  });

  test("I-13 anonymous request is rejected @smoke", async ({ anonApi }) => {
    expect((await anonApi.orders.get("any-id")).status()).toBe(401);
  });
});
```

Production smoke rows read records that already exist (IDs from settings), never create them — a test that needs a fixture which creates data is `writes`.

## 6. Side effects and cleanup

- Assert side effects through the API (follow-up GET), or through the DB in integration tests — not by trusting the write's response alone.
- Every created resource is deleted in teardown, or created under a test-scoped tenant/user removed afterwards.
- Unique identifiers per test (factories handle this — `references/test-data.md`) so parallel runs never collide.

## 7. Access control

- One `<role>_api` fixture per role keeps access-control rows readable.
- Multi-tenant APIs: always include one cross-tenant read and one cross-tenant write that must be refused.

## 8. Timeouts

Set an explicit client timeout (10 s in the fixtures above) so a hung service fails the test instead of the job. Latency is asserted only in the performance phase; functional API tests assert correctness.
