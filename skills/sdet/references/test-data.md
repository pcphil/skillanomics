# Test data and environments

Loaded at `[SDET: Data]`. Shared, mutable, order-dependent data is the largest source of flaky suites. Every test creates what it needs, uses unique values, and cleans up after itself.

## 1. Rules

1. **Each test owns its data.** Create it in a fixture; delete it in teardown.
2. **Unique values per test.** Emails, usernames, SKUs carry a UUID or worker-scoped suffix so parallel runs never collide.
3. **Defaults sane, override what matters.** A factory call states only the fields the test cares about.
4. **Deterministic where behavior depends on it.** Seed Faker; freeze time.
5. **Real engines for integration.** Run the production database engine in a container, not a lighter stand-in.

## 2. Factories

### Python (factory_boy + Faker)

```python
# tests/factories/user.py
from uuid import uuid4
import factory
from app.models import User

factory.random.reseed_random(1234)

class UserFactory(factory.alchemy.SQLAlchemyModelFactory):
    class Meta:
        model = User
        sqlalchemy_session_persistence = "flush"   # bind the session in a fixture: UserFactory._meta.sqlalchemy_session = db_session

    email = factory.LazyFunction(lambda: f"user-{uuid4().hex[:8]}@example.test")
    name = factory.Faker("name")
    role = "member"
```

Simpler and equally valid when no ORM factory integration is needed:

```python
# tests/factories/user.py
from uuid import uuid4
from faker import Faker

fake = Faker()
Faker.seed(1234)

def user_payload(**overrides):
    return {
        "email": f"user-{uuid4().hex[:8]}@example.test",
        "name": fake.name(),
        "role": "member",
        **overrides,
    }
```

Fixture that creates via the API and cleans up:

```python
# tests/conftest.py
import pytest
from tests.factories.user import user_payload

@pytest.fixture
def user(api):
    resp = api.post("/users", json=user_payload())
    assert resp.status_code == 201
    created = resp.json()
    yield created
    api.delete(f"/users/{created['id']}")
```

### TS (@faker-js/faker)

```ts
// tests/factories/user.ts
import { faker } from "@faker-js/faker";
import { randomUUID } from "node:crypto";

faker.seed(1234);

export function userPayload(overrides: Partial<UserInput> = {}): UserInput {
  return {
    email: `user-${randomUUID().slice(0, 8)}@example.test`,
    name: faker.person.fullName(),
    role: "member",
    ...overrides,
  };
}
```

```ts
// tests/e2e/fixtures.ts (excerpt) — Playwright fixture with teardown
export const test = base.extend<{ user: User }>({
  user: async ({ request }, use) => {
    const res = await request.post("/users", { data: userPayload() });
    const user = await res.json();
    await use(user);
    await request.delete(`/users/${user.id}`);
  },
});
```

Use the reserved `.test` domain (or `example.com`) for generated emails so nothing is ever delivered.

## 3. Containers (testcontainers)

Use for integration tests that need a real DB, cache, or broker. Requires Docker on the machine and in CI — ask the user before adding it.

### Python

```python
# tests/conftest.py
import pytest
from testcontainers.postgres import PostgresContainer
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="session")
def pg_url():
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg.get_connection_url()

@pytest.fixture(scope="session")
def engine(pg_url):
    engine = create_engine(pg_url)
    run_migrations(engine)          # the project's own migration entry point
    return engine

@pytest.fixture
def db_session(engine):
    conn = engine.connect()
    tx = conn.begin()
    session = sessionmaker(bind=conn)()
    yield session
    session.close()
    tx.rollback()                    # every test leaves the DB as it found it
    conn.close()
```

### TS

```ts
// tests/integration/setup.ts — Vitest globalSetup
import { PostgreSqlContainer, type StartedPostgreSqlContainer } from "@testcontainers/postgresql";

let container: StartedPostgreSqlContainer;

export async function setup() {
  container = await new PostgreSqlContainer("postgres:16-alpine").start();
  process.env.DATABASE_URL = container.getConnectionUri();
  await runMigrations(process.env.DATABASE_URL); // the project's own migration entry point
}

export async function teardown() {
  await container.stop();
}
```

Pin image tags to the version production runs.

## 4. Seed data vs. per-test data

- **Reference data** (countries, plans, feature flags) — seed once per session via migrations or a seed script; read-only in tests.
- **Entity data** (users, orders) — per test, via factories and fixtures.
- **Staging** — never depend on data created by hand; create and clean up through the API.
- **Production** — read-only smoke only: read pre-existing, known-stable records (IDs supplied through env vars). Create nothing.

## 5. Environments

- Local: app via the project's own run command or `docker compose up`; tests point at it via `BASE_URL`.
- Staging: base URL and credentials from CI secrets.
- One suite, many environments: only env vars change between targets — the code is identical.
