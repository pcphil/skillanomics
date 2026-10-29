# Performance tests (Locust)

Loaded at `[SDET: Perf]`. Locust drives HTTP load from Python and is used on every stack — load tests are black-box, so the app's language does not matter.

**Targets: local or staging only.** Load never runs against production. Confirm the target URL with the user and confirm staging capacity or rate limits before the first run.

**Vendor-hosted products (SaaS):** off by default. Run only against a vendor sandbox, after the user confirms the vendor's written approval, within its limits (`references/vendor-apps.md` §8).

## 1. What to measure

From the strategy, for each hot path:

| Metric | Example threshold |
|--------|-------------------|
| p95 latency | < 500 ms |
| p99 latency | < 1200 ms |
| Error rate | < 1 % |
| Throughput | ≥ 50 req/s sustained |

Thresholds come from an SLO, a product requirement, or a baseline run — ask the user which. Without one, run a baseline first and propose thresholds from it.

## 2. Load profiles

| Profile | Shape | Purpose |
|---------|-------|---------|
| Smoke | 1–5 users, 1 min | script works, endpoints reachable |
| Load | expected peak users, 10–15 min | thresholds hold at normal peak |
| Stress | ramp past peak until errors | find the breaking point (staging only, with approval) |
| Soak | expected load, 1 h+ | leaks, connection exhaustion |

Default to Smoke + Load. Stress and Soak only when the strategy asks.

## 3. Locustfile

```python
# tests/perf/locustfile.py
import os
from uuid import uuid4
from locust import HttpUser, task, between, events

TEST_API_TOKEN = os.environ.get("TEST_API_TOKEN", "")

class ShopUser(HttpUser):
    wait_time = between(1, 3)

    def on_start(self):
        self.client.headers["Authorization"] = f"Bearer {TEST_API_TOKEN}"

    @task(5)
    def browse_products(self):
        with self.client.get("/products?page=1", name="GET /products", catch_response=True) as resp:
            if resp.status_code != 200 or not resp.json().get("items"):
                resp.failure(f"bad products response: {resp.status_code}")

    @task(2)
    def view_product(self):
        self.client.get("/products/sku-1001", name="GET /products/:sku")

    @task(1)
    def create_cart(self):
        with self.client.post("/carts", json={"ref": f"perf-{uuid4().hex[:8]}"}, name="POST /carts", catch_response=True) as resp:
            if resp.status_code != 201:
                resp.failure(f"cart create failed: {resp.status_code}")
```

- `@task(weight)` mirrors real traffic mix — derive weights from analytics or logs when available.
- `name=` groups parameterized URLs so stats aggregate per route.
- `catch_response=True` validates bodies, not just status codes — a fast 200 with an error body is a failure.
- Data created under load carries a `perf-` prefix so it can be cleaned up in bulk afterwards.

## 4. Thresholds as pass/fail

Locust exits 0 by default even when slow. Fail the run when thresholds break:

```python
# tests/perf/locustfile.py (continued)
P95_MS = int(os.environ.get("PERF_P95_MS", "500"))
MAX_FAIL_RATIO = float(os.environ.get("PERF_MAX_FAIL_RATIO", "0.01"))

@events.quitting.add_listener
def enforce_thresholds(environment, **_kwargs):
    total = environment.stats.total
    if total.fail_ratio > MAX_FAIL_RATIO:
        environment.process_exit_code = 1
        print(f"FAIL: error ratio {total.fail_ratio:.2%} > {MAX_FAIL_RATIO:.2%}")
    elif total.get_response_time_percentile(0.95) > P95_MS:
        environment.process_exit_code = 1
        print(f"FAIL: p95 {total.get_response_time_percentile(0.95)} ms > {P95_MS} ms")
```

## 5. Running

```bash
# local smoke (headless)
locust -f tests/perf/locustfile.py --host "$TEST_BASE_URL" --headless -u 5 -r 1 -t 1m \
  --html tests/reports/perf.html --csv tests/reports/perf

# staging load
locust -f tests/perf/locustfile.py --host "$TEST_BASE_URL" --headless -u 200 -r 10 -t 15m \
  --html tests/reports/perf.html --csv tests/reports/perf
```

Before running, check that `$TEST_BASE_URL` is a local or staging host. If it resolves to production, stop and tell the user.

## 6. In CI

Perf runs are not on the merge gate. Run them on a schedule or manual trigger against staging — see `references/ci-github-actions.md`. Upload `tests/reports/perf.html` and the CSVs as artifacts.

## 7. Reading results

Report per route: requests, failures, median, p95, p99, req/s — and which thresholds passed or failed. Flag any route whose p99 is more than ~3× its median as a latency-spike suspect.
