# Scaffold: TS

Loaded at `[SDET: Scaffold]` after `references/scaffold-plan.md`, when the suite or a layer is JS/TS.

## 1. Layout

```
tests/
├── TESTING.md
├── .env.example  .env.local  .env.staging  .env   # same convention as Python
├── .auth/  reports/                 # git-ignored
├── playwright.config.ts             # only when the repo has none (§4); standalone: at the root
├── vitest.config.ts                 # only when the repo has no Vitest/Vite config
├── settings.ts
├── support/{clients/,models/}
├── factories/
├── unit/                           # *.test.ts (Vitest); or co-located src/**/*.test.ts
├── api/
│   └── fixtures.ts                 # role-based API fixtures
├── e2e/
│   ├── fixtures.ts                 # page objects, extends api fixtures
│   ├── auth.setup.ts               # per-role storage state
│   └── pages/
└── perf/locustfile.py              # Locust is Python on every stack
package.json                         # + devDependencies, test:* scripts
```

Add to `.gitignore` (merge): `tests/.env`, `tests/.auth/`, `tests/reports/`. Standalone repo: `.env`, `.auth/`, `reports/` at its root — read `tests/.auth/` and `tests/reports/` in this skill as those.

## 2. Settings (dotenv + zod)

Same files, same `TEST_` names, same precedence. TS settings are validated when first imported.

```ts
// tests/settings.ts
import path from "node:path";
import dotenv from "dotenv";
import { z } from "zod";

const envDir = path.resolve("tests"); // standalone repo: path.resolve(".")
dotenv.config({ path: path.join(envDir, ".env") }); // never overrides exported vars
const targetEnv = process.env.TEST_TARGET_ENV ?? "local";
dotenv.config({ path: path.join(envDir, `.env.${targetEnv}`) });

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`${name} missing for ${targetEnv} — see tests/.env.example`);
  return value;
}

// `KEY=` in an env file loads as "", which counts as unset
const blank = (schema: z.ZodTypeAny) => z.preprocess((v) => (v === "" ? undefined : v), schema);

const env = z
  .object({
    TEST_TARGET_ENV: z.enum(["local", "staging", "production"]),
    TEST_BASE_URL: z.string().url(),
    TEST_API_URL: blank(z.string().url().optional()),
    TEST_ROLES: blank(z.string().default("member")),
    TEST_DEFAULT_ROLE: blank(z.string().default("member")),
    TEST_LOGIN_PATH: blank(z.string().default("/auth/login")),
    TEST_PRODUCTION_HOSTS: blank(z.string().default("")),
  })
  .parse({ ...process.env, TEST_TARGET_ENV: targetEnv });

export const roleNames = (env.TEST_ROLES as string).split(",").map((r) => r.trim()).filter(Boolean);

// Lazy, like Python's Settings.role(): only the roles a run uses need credentials
export function credentials(role: string) {
  const key = role.toUpperCase();
  return {
    username: required(`TEST_ROLE_${key}_USERNAME`),
    password: required(`TEST_ROLE_${key}_PASSWORD`),
    totpSecret: process.env[`TEST_ROLE_${key}_TOTP_SECRET`], // vendor SSO with MFA
  };
}

export const settings = {
  targetEnv: env.TEST_TARGET_ENV,
  baseUrl: env.TEST_BASE_URL,
  apiBase: (env.TEST_API_URL as string | undefined) || env.TEST_BASE_URL,
  defaultRole: env.TEST_DEFAULT_ROLE as string,
  loginPath: env.TEST_LOGIN_PATH as string,
};

// Production guard: a production host labelled as another env fails before any test runs
const prodHosts = (env.TEST_PRODUCTION_HOSTS as string).split(",").map((h) => h.trim()).filter(Boolean);
for (const url of [settings.baseUrl, settings.apiBase]) {
  if (prodHosts.includes(new URL(url).hostname) && settings.targetEnv !== "production") {
    throw new Error(`${url} is a production host but TEST_TARGET_ENV=${settings.targetEnv}`);
  }
}
```

TS production guard:

```ts
// playwright.config.ts (excerpt)
grep: settings.targetEnv === "production" ? /@smoke/ : undefined,
grepInvert: settings.targetEnv === "production" ? /@writes/ : undefined,
```

## 3. Vitest (unit only)

An existing Vitest or Vite config gets a `test` block. Without one, create `tests/vitest.config.ts` and run it with `vitest run --config tests/vitest.config.ts`:

```ts
// tests/vitest.config.ts — paths resolve from the repo root, where the scripts run
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["tests/unit/**/*.test.ts", "src/**/*.test.ts"],
    reporters: ["default", "junit"],
    outputFile: { junit: "tests/reports/junit-unit.xml" },
  },
});
```

## 4. Playwright Test (API + UI)

An existing `playwright.config.*` means the framework exists: extend it. Without one, create `tests/playwright.config.ts` (standalone: at the root, importing `./tests/settings`, with `.auth` and `reports` at the root) and run it through `package.json` scripts:

```json
"test:unit": "vitest run --config tests/vitest.config.ts",
"test:api": "playwright test -c tests/playwright.config.ts --project=api",
"test:e2e": "playwright test -c tests/playwright.config.ts --project=chromium"
```

```ts
// tests/playwright.config.ts — paths resolve from the repo root, where the scripts run (as in settings.ts)
import path from "node:path";
import { defineConfig, devices } from "@playwright/test";
import { settings } from "./settings";

const authDir = path.resolve("tests/.auth");
const reportDir = path.resolve("tests/reports");

const e2e = (name: string, device: object) => ({
  name,
  testMatch: /e2e\/.*\.spec\.ts/,
  use: { ...device, storageState: path.join(authDir, `${settings.defaultRole}.json`) },
  dependencies: ["setup"],
});

export default defineConfig({
  testDir: ".",
  fullyParallel: true,
  retries: process.env.CI ? 1 : 0,
  reporter: [
    ["list"],
    ["junit", { outputFile: path.join(reportDir, "junit-pw.xml") }],
    ["html", { open: "never", outputFolder: path.join(reportDir, "html") }],
  ],
  use: {
    baseURL: settings.baseUrl,
    trace: "on-first-retry",
    screenshot: "only-on-failure",
  },
  projects: [
    { name: "api", testMatch: /api\/.*\.spec\.ts/, use: { baseURL: settings.apiBase } },
    { name: "setup", testMatch: /e2e\/auth\.setup\.ts/ },
    e2e("chromium", devices["Desktop Chrome"]),
    // nightly only — references/ci-github-actions.md
    e2e("firefox", devices["Desktop Firefox"]),
    e2e("webkit", devices["Desktop Safari"]),
    e2e("mobile", devices["Pixel 7"]),
  ],
});
```

Merge gate runs `--project=api --project=chromium`; nightly adds `firefox`, `webkit`, `mobile`. When the strategy flags a browser as high-risk (e.g. a large Safari audience), move it onto the gate. Retries apply once, in CI only.

## 5. Proof tests

```ts
// tests/api/smoke.spec.ts
import { test, expect } from "./fixtures";

test("I-00 health endpoint is up and matches schema @smoke", async ({ anonApi }) => {
  const res = await anonApi.health.get();
  expect(res.status()).toBe(200);
  await anonApi.health.getOk(); // validates schema
});
```

```ts
// tests/e2e/smoke.spec.ts
import { test, expect } from "./fixtures";

test.use({ storageState: { cookies: [], origins: [] } }); // anonymous

test("E-00 home page loads @smoke", async ({ homePage }) => {
  await homePage.open();
  await expect(homePage.mainHeading).toBeVisible();
});
```
