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
# tests/support/fixtures/data.py
import pytest
from tests.factories.user import user_payload

@pytest.fixture
def user(admin_api):
    created = admin_api.users.create_ok(user_payload())  # asserts 2xx + schema, returns the model
    yield created
    admin_api.users.delete(created.id)
```

Fixtures create data through the API client layer (`references/api.md`), so setup failures surface at the setup line with the status and body.

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
// tests/e2e/fixtures.ts (excerpt) — Playwright fixture with teardown, built on the API fixtures
export const test = apiTest.extend<{ user: User }>({
  user: async ({ adminApi }, use) => {
    const user = await adminApi.users.createOk(userPayload());
    await use(user);
    await adminApi.users.delete(user.id);
  },
});
```

Use the reserved `.test` domain (or `example.com`) for generated emails so nothing is ever delivered.

## 3. Containers (testcontainers)

Use for integration tests that need a real DB, cache, or broker. Requires Docker on the machine and in CI — ask the user before adding it.

### Python

Under xdist every worker runs its own copy of each session fixture, so a per-worker container would mean N containers. Instead: **one database server for the run, one database per worker on it.**

- **Parallel runs (`-n`)** — the server is started outside pytest and passed in as `TEST_DATABASE_URL`: a CI service container (`references/ci-github-actions.md` §2) or `docker compose up -d db` locally. A server started outside pytest has a clean lifecycle; one started by the first worker can be reaped when that worker exits while others still use it.
- **Single-process runs** — with `TEST_DATABASE_URL` unset, the fixture starts a testcontainer itself, so `pytest tests/integration` works on a fresh clone with only Docker.

```python
# tests/support/fixtures/data.py — registered via pytest_plugins
import pytest
from sqlalchemy import create_engine, make_url, text
from sqlalchemy.orm import sessionmaker

@pytest.fixture(scope="session")
def database_server(env_settings, worker_id):
    """One Postgres server for the whole run."""
    if env_settings.database_url:
        yield env_settings.database_url
        return
    if worker_id != "master":
        raise RuntimeError(
            "TEST_DATABASE_URL must be set when running with -n: start one server "
            "(docker compose up -d db, or a CI service container) and export its URL"
        )
    from testcontainers.postgres import PostgresContainer
    with PostgresContainer("postgres:16-alpine") as pg:
        yield pg.get_connection_url()

@pytest.fixture(scope="session")
def engine(database_server, worker_id):
    """One database per xdist worker, so workers never share rows."""
    name = f"test_{worker_id}"
    admin = create_engine(database_server, isolation_level="AUTOCOMMIT")
    with admin.connect() as conn:
        conn.execute(text(f'DROP DATABASE IF EXISTS "{name}" WITH (FORCE)'))
        conn.execute(text(f'CREATE DATABASE "{name}"'))
    engine = create_engine(make_url(database_server).set(database=name))
    run_migrations(engine)          # the project's own migration entry point
    yield engine
    engine.dispose()
    with admin.connect() as conn:
        conn.execute(text(f'DROP DATABASE IF EXISTS "{name}" WITH (FORCE)'))
    admin.dispose()

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
- **Vendor sandbox** — shared with other teams: prefix every created record with the run ID and sweep leftovers nightly (`references/vendor-apps.md` §9).
- **Production** — read-only smoke only: read pre-existing, known-stable records (IDs supplied through env vars). Create nothing.

## 5. Environments

- Local: app via the project's own run command or `docker compose up`; tests point at it via `TEST_BASE_URL`.
- Staging: base URL and credentials from CI secrets.
- One suite, many environments: only env vars change between targets — the code is identical.
