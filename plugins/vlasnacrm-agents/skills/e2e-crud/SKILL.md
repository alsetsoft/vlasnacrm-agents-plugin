---
name: e2e-crud
description: Playwright end-to-end tests for a data app — one-time setup (config, auth, safe test environment, CI), how to write CRUD, relationship and regression tests whose titles and steps read as real test cases for QA (they feed the test-docs PDF). Load when adding e2e tests for a feature/entity, writing a regression test for a bug fix, or setting up e2e/CI in a project.
---

# E2E tests (Playwright)

Tests are the **single source of truth** for what is verified: their titles, steps, tags and annotations are turned into the testers' document by the `test-docs` skill. Write them so a tester can read them without opening the code.

## Safety first

- Tests run against a **test environment** by default: a separate test database/project (e.g. a free Supabase project "test") or a local stack — never the team's shared working database. Production is reachable only through the two opt-in, zero-residue paths in *Bug-fix verification on production* (DB tests that always roll back; `@prod-safe` UI tests with ledger cleanup) — never by pointing the normal suite at it.
- `playwright.config.ts` refuses to start when `E2E_BASE_URL` points to a forbidden host (`E2E_FORBIDDEN_HOSTS`, comma-separated, set in CI and `.env.e2e.example`).
- A dedicated test user (`E2E_EMAIL` / `E2E_PASSWORD`) with normal (not admin) rights unless a test needs admin; credentials only from env/CI secrets, never committed.
- Every record a test creates has a unique prefix (`e2e-<runId>-…`) and is **deleted** after the run (not archived — archived rows are still rows) — see *Test data lifecycle*. A run ends with a residue check; leftover test data fails the run.

## Setup (once per project)

```
e2e/
  auth.setup.ts        # logs in once, saves storageState
  fixtures.ts          # uniq(), api client as the test user, per-test cleanup
  global-setup.ts      # e2e_cleanup(): leftovers of a crashed run
  global-teardown.ts   # e2e_cleanup() + e2e_residue() — fails the run on leftovers
  cleanup.md           # per entity: what its triggers touch and how the sweep reverts it
  db/fixtures.ts       # rollback-only DB fixture (see Bug-fix verification on production)
  db/<module>.db.spec.ts
  relations.yaml       # business flows for the testers' document (see below)
  <module>.spec.ts     # one file per module/entity
playwright.config.ts
.env.e2e.example       # E2E_BASE_URL, E2E_EMAIL, E2E_PASSWORD, E2E_FORBIDDEN_HOSTS, E2E_DB_URL, E2E_USER_ID, E2E_TARGET, E2E_ALLOW_PROD
supabase/migrations/<ts>_e2e_support.sql   # e2e_accounts, e2e_created, e2e_track, e2e_cleanup, e2e_residue
```

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

import fs from "node:fs";

const baseURL = process.env.E2E_BASE_URL ?? "http://localhost:3000";
const prod = process.env.E2E_TARGET === "prod";
if (prod) {                                                           // bug-fix verification on production: @prod-safe only
  const policy = JSON.parse(fs.readFileSync(".vlasnacrm/policy.json", "utf8"));
  if (process.env.E2E_ALLOW_PROD !== "1" || policy.e2e?.prod !== true)
    throw new Error("E2E refused: production needs policy.e2e.prod = true and E2E_ALLOW_PROD=1");
} else {
  const forbidden = (process.env.E2E_FORBIDDEN_HOSTS ?? "").split(",").map((h) => h.trim()).filter(Boolean);
  if (forbidden.some((h) => new URL(baseURL).host.endsWith(h))) throw new Error(`E2E refused: ${baseURL} is a forbidden host`);
}

export default defineConfig({
  testDir: "./e2e",
  globalSetup: "./e2e/global-setup.ts",
  globalTeardown: "./e2e/global-teardown.ts",
  ...(prod ? { grep: /@prod-safe/, workers: 1 } : {}),
  fullyParallel: !prod,
  retries: process.env.CI && !prod ? 1 : 0,
  reporter: process.env.CI ? [["list"], ["json", { outputFile: "test-results/results.json" }], ["html", { open: "never" }]] : "list",
  use: { baseURL, trace: "retain-on-failure", screenshot: "only-on-failure", locale: "uk-UA", timezoneId: "Europe/Kyiv" }, // from policy.locale / policy.timezone
  projects: [
    { name: "setup", testMatch: /auth\.setup\.ts/ },
    { name: "db", testMatch: /\.db\.spec\.ts/ },                     // rollback-only DB tests, no browser
    { name: "chromium", testIgnore: /\.db\.spec\.ts/, use: { ...devices["Desktop Chrome"], viewport: { width: 1366, height: 768 }, storageState: "e2e/.auth/user.json" }, dependencies: ["setup"] },
  ],
  webServer: process.env.E2E_BASE_URL ? undefined : { command: "npm run dev", url: baseURL, reuseExistingServer: true, timeout: 120_000 },
});
```

```ts
// e2e/auth.setup.ts
import { test as setup, expect } from "@playwright/test";
setup("login", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel(/e-?mail|пошта/i).fill(process.env.E2E_EMAIL!);
  await page.getByLabel(/пароль|password/i).fill(process.env.E2E_PASSWORD!);
  await page.getByRole("button", { name: /увійти|sign in|log in/i }).click();
  await expect(page).not.toHaveURL(/\/login/);
  await page.context().storageState({ path: "e2e/.auth/user.json" });
});
```

- `package.json`: `"e2e": "playwright test"`, `"e2e:smoke": "playwright test --grep @smoke"`; dev dependency `@playwright/test`.
- `.gitignore`: `e2e/.auth/`, `test-results/`, `playwright-report/`, `.env.e2e`.
- Data helpers in `fixtures.ts`: `uniq(name)` → `e2e-${runId}-${name}`; an API client signed in as the test user (same RLS as the UI) to create **prerequisites** quickly; a `created` fixture that collects `{table, id}` (prerequisites from the API result, the entity under test from the UI's insert response or its URL) and deletes them in teardown. The entity under test is created **through the UI**. Dev dependency `pg` for DB tests.

## Test data lifecycle (every environment)

Test data must leave **nothing behind** — not in the test DB (tests stay independent and lists stay small), and above all not in production. Three layers, each one catching what the previous missed:

1. **Per test** — the fixture registers every record it creates (prerequisites through the API, the entity under test from the UI response) and deletes them in its teardown, children first. Teardown runs even when the test fails (`use()` in a fixture, not code after the assertions).
2. **Per run** — `globalTeardown` calls the server-side sweep (`e2e_cleanup()`): deletes **everything the dedicated e2e account created**, in FK order, including rows created by triggers in the same request (audit, journal, movement rows — they carry the same `auth.uid()`). `globalSetup` calls it too, so a crashed previous run is cleaned before the next one starts.
3. **Residue check** — `e2e_residue()` returns what is still there (table, id); the teardown fails the run with that list when it's not empty. Never "warn and continue".

Rules that make this possible:

- **The e2e account owns test data.** A dedicated user (`E2E_EMAIL`) that real people never use; "created by this account" = test data, nothing else needs guessing. Prefix `e2e-…` stays for humans (visible in lists) and as a second key.
- **Own the whole graph.** A test creates every record it writes to — its own customer, product, warehouse/store when the flow moves stock — so side effects land on test-owned rows and disappear with them. A test that has to change a pre-existing record (shared settings, a real product's stock, a real document) is **not** prod-safe, and in the test DB it must restore the old value in teardown.
- **Archive-only apps still delete test data.** When the DB forbids DELETE (archive-only triggers, append-only audit), the sweep is a `SECURITY DEFINER` function that sets a transaction-local flag (`set local app.e2e_cleanup = 'on'`) which those triggers honour **only together with** the e2e account check — the bypass is unusable for anyone else.
- **Side effects are listed and reverted.** For each tested entity write down what its triggers touch (stock/balances/totals, audit/history, counters, caches) in `e2e/cleanup.md`. Rows the e2e account inserted are swept; aggregates on test-owned rows go with them; anything else must be recalculated by the sweep (call the same recalc function the app uses) — or the test can't be prod-safe. Sequence/number gaps (document numbers consumed by a test) are the one accepted residue: say so in `cleanup.md`.
- **No outward effects.** Tests never send real e-mails/SMS/webhooks/payments or sync to external systems: stub those calls with `page.route()` or a test-mode flag; a feature that can't be stubbed is never `@prod-safe`.

```sql
-- migration: e2e support (safe to apply to production — inert unless the e2e account acts)
create table if not exists e2e_accounts (user_id uuid primary key references auth.users on delete cascade);
create table if not exists e2e_created (tbl regclass not null, id uuid not null, created_at timestamptz default now(),
  primary key (tbl, id));
alter table e2e_accounts enable row level security;  -- no policies: invisible to clients
alter table e2e_created enable row level security;

create or replace function is_e2e_account() returns boolean language sql stable security definer set search_path = public as
$$ select exists (select 1 from e2e_accounts where user_id = auth.uid()) $$;

-- AFTER INSERT on every table a test can create rows in (entities, their lines, trigger-made rows: audit, journals…)
create or replace function e2e_track() returns trigger language plpgsql security definer set search_path = public as $$
begin
  if is_e2e_account() then insert into e2e_created(tbl, id) values (TG_RELID::regclass, new.id) on conflict do nothing; end if;
  return null;
end $$;
-- create trigger zz_e2e_track after insert on <table> for each row execute function e2e_track();

-- archive-only / append-only guards add one line at the top:
--   if current_setting('app.e2e_cleanup', true) = 'on' and is_e2e_account() then return old; end if;

create or replace function e2e_cleanup() returns integer language plpgsql security definer set search_path = public as $$
declare r record; n integer := 0; k integer; i integer;
begin
  if not is_e2e_account() then raise exception 'e2e_cleanup: not an e2e account' using errcode = '42501'; end if;
  perform set_config('app.e2e_cleanup', 'on', true);
  for i in 1..5 loop                               -- FK order without hand-maintaining it: retry what a child still blocks
    for r in select tbl, array_agg(id) ids from e2e_created group by tbl loop
      begin
        execute format('delete from %s where id = any($1)', r.tbl) using r.ids;
        get diagnostics k = row_count; n := n + k;
        delete from e2e_created where tbl = r.tbl;
      exception when foreign_key_violation then null; -- a child row still points here; next pass
      end;
    end loop;
    exit when not exists (select 1 from e2e_created);
  end loop;
  -- + recalc functions for aggregates on rows the e2e account didn't create (see e2e/cleanup.md), if any
  return n;
end $$;

create or replace function e2e_residue() returns table(tbl text, id uuid) language sql security definer set search_path = public as
$$ select tbl::text, id from e2e_created where is_e2e_account() $$;

revoke all on function e2e_cleanup(), e2e_residue() from public, anon;
grant execute on function e2e_cleanup(), e2e_residue() to authenticated;  -- the function itself checks the account
-- once, by an admin:  insert into e2e_accounts select id from auth.users where email = '<E2E_EMAIL>';
```

Tables with non-`uuid` keys: store `id::text` and cast in the delete. Rows that cascade-delete with their parent don't need tracking but tracking them is harmless.

```ts
// e2e/global-setup.ts and e2e/global-teardown.ts (both: sign in as the e2e account with the API client)
const { error } = await api.rpc("e2e_cleanup");                       // setup: leftovers of a crashed run
if (error) throw new Error(`e2e cleanup failed: ${error.message}`);
// teardown only, after e2e_cleanup:
const { data: left } = await api.rpc("e2e_residue");
if (left?.length) throw new Error(`e2e residue — test data left in the DB:\n${left.map((r) => `${r.tbl} ${r.id}`).join("\n")}`);
```

Wire them with `globalSetup` / `globalTeardown` in `playwright.config.ts`. The per-test fixture deletes through the same API (allowed by the e2e-account bypass) so a long suite doesn't pile up data until the end.

## Bug-fix verification on production (opt-in, zero residue)

Sometimes a bug only shows on production data, or a hotfix goes straight to prod and must be proven there. Two paths, in order of preference. Both need `policy.e2e.prod: true` **and** `E2E_ALLOW_PROD=1` in the shell of the person running them; agents never set that variable themselves — they ask.

### 1. DB regression test — always rolled back (preferred)

For anything the database decides (RPC/function results, triggers, constraints, RLS, totals, limits) — most "wrong number / wrong rule" bugs. The test runs as the real user role inside **one transaction that is always rolled back**, so it can't leave data even if it crashes mid-way (the connection drops → Postgres rolls back).

```ts
// e2e/db/fixtures.ts — Playwright project "db" (no browser, no auth.setup dependency)
import { test as base } from "@playwright/test";
import pg from "pg";

// statements that end/escape the transaction — checked at statement start, outside quotes/dollar-quotes/comments,
// so `case … end` and `do $$ begin … end $$` still work (COMMIT inside a DO block fails on its own inside a transaction) …
const outsideQuotes = (q: string) => q.replace(/\$(\w*)\$[\s\S]*?\$\1\$|'(?:[^']|'')*'|--[^\n]*|\/\*[\s\S]*?\*\//g, " ");
const TX_ESCAPE = /(^|;)\s*(commit|end|begin|abort|start\s+transaction|rollback(?!\s+to)|prepare\s+transaction|set\s+(session\s+)?role|reset\s+role)\b/i;
// … and calls whose effect a rollback can't undo
const NON_TX = /\b(vacuum|reindex|cluster|alter\s+system|dblink\w*|pg_terminate_backend|pg_cancel_backend|lo_export|http_(get|post|put|patch|delete)|net\.http_\w+)\b|\bcopy\b[^;]*\bprogram\b/i;

export const test = base.extend<{ db: (q: string, params?: unknown[]) => Promise<any[]>; asUser: (email: string) => Promise<void> }>({
  db: async ({}, use) => {
    const client = new pg.Client({ connectionString: process.env.E2E_DB_URL, application_name: "e2e-db-rollback" });
    await client.connect();
    await client.query("begin");
    await client.query("set local statement_timeout = '15s'; set local lock_timeout = '2s'; set local idle_in_transaction_session_timeout = '60s'");
    try {
      await use(async (q, params) => {
        if (TX_ESCAPE.test(outsideQuotes(q)) || NON_TX.test(q)) throw new Error(`db test refused a statement that can escape the rollback: ${q.slice(0, 80)}`);
        return (await client.query(q, params)).rows;
      });
    } finally {
      await client.query("rollback").catch(() => {});                  // ALWAYS — pass or fail
      await client.end();
    }
  },
  asUser: async ({ db }, use) => {                                    // run as the e2e account: same RLS/permission checks as the UI
    await use(async (email) => {                                      // E2E_USER_ID = auth.users.id of E2E_EMAIL (the runner role can't read auth.users)
      await db("select set_config('request.jwt.claims', $1, true)", [JSON.stringify({ sub: process.env.E2E_USER_ID, email, role: "authenticated" })]);
      await db("select set_config('role', 'authenticated', true)");  // = set local role; plain `set role` is refused by the guard
    });
  },
});
```

```ts
// e2e/db/orders.db.spec.ts
test("Сума замовлення в списку = сума в картці з ПДВ «зверху»: обидві 120,00", {
  tag: ["@regression", "@orders", "@db", "@prod-safe"],
  annotation: [{ type: "bug", description: "PRJ-42: у списку сума була без ПДВ" }],
}, async ({ db, asUser }) => {
  await asUser(process.env.E2E_EMAIL!);
  const [o] = await db("insert into orders(...) values (...) returning id");   // own data, rolled back at the end
  const [row] = await db("select total from order_totals where id = $1", [o.id]);
  expect(Number(row.total), "сума в списку має включати ПДВ «зверху»").toBe(120);
});
```

- Connection: `E2E_DB_URL` — a dedicated login role (`e2e_runner`: `login`, `grant authenticated to e2e_runner`, nothing else), never the owner/service credentials. The guard regex is a seatbelt against escaping the transaction; the role is the real limit.
- What doesn't roll back, and is therefore refused or avoided: `COMMIT`/`END`/top-level `ROLLBACK`, `SET ROLE` outside `asUser` (switching back to the runner role mid-test), synchronous HTTP from SQL, `dblink`, admin commands. Sequence values do advance (number gap only).
- Keep it short: the transaction holds row locks on what it touched until the rollback; `lock_timeout` makes the test fail instead of blocking real users.
- Read-only checks of real data (`select … from <real table>`) are fine and often the point (reproduce on the record from the bug report); writes go only to the test's own rows.

### 2. `@prod-safe` UI regression test — ledger cleanup

When the bug is in the UI flow itself (a button, a redirect, a double submit, a form that sends the wrong payload) and can't be a DB test. Rules on top of *Test data lifecycle*:

- Only tests tagged `@prod-safe` run: `E2E_TARGET=prod` makes the config add `grep: /@prod-safe/`, `workers: 1`, `retries: 0`, and refuse to start without `E2E_ALLOW_PROD=1` and `policy.e2e.prod`. A CLI `--grep` replaces the config's `grep`, so both fixture files also carry an auto fixture that fails any test without the tag:

  ```ts
  prodGuard: [async ({}, use, testInfo) => {
    if (process.env.E2E_TARGET === "prod" && !testInfo.tags.includes("@prod-safe"))
      throw new Error(`refused on production: "${testInfo.title}" is not @prod-safe`);
    await use();
  }, { auto: true }],
  ```
  The forbidden-host guard stays for every other target.
- A test may be tagged `@prod-safe` only if: it owns its whole graph; every table it (or its triggers) inserts into has the `e2e_track` trigger; its side effects are listed in `cleanup.md` and reverted by the sweep; outward calls are stubbed; it is idempotent (a re-run after a crash is fine).
- The e2e account on prod has the minimal role the test needs (not admin unless the bug is admin-only), its own password in the runner's env, and `e2e_accounts` lists it.
- Run it off-hours: for the seconds it runs, `e2e-…` records are visible to real users in lists and counters.
- After the run: the residue check passed (the run fails otherwise) — quote it in the report. If it failed, stop and report the residue list; don't re-run blindly.

## How a test reads (it becomes a test case)

```ts
test.describe("Замовлення покупців", () => {                       // module = the UI section name
  test("Створити замовлення з 2 позиціями: після «Зберегти» лишаємось у картці, сума = Σ кількість × ціна", {
    tag: ["@orders", "@smoke"],                                      // @<entity table> + @smoke for the core path
    annotation: [{ type: "relation", description: "orders → order_items (1:N)" }],
  }, async ({ page }) => {
    await test.step("Відкрити «Замовлення» → «+ Замовлення»", async () => { … });
    await test.step("Обрати контрагента e2e-…, додати 2 товари з кількістю 2 і 3", async () => { … });
    await test.step("Натиснути «Зберегти»", async () => { … });
    await test.step("URL /orders/<id>, тост «Збережено», сума = 2×ціна₁ + 3×ціна₂", async () => {
      await expect(page, "після створення має відкритись картка нового замовлення").toHaveURL(/\/orders\/[\w-]+$/);
    });
  });
});
```

- **Title** = `<action>: <expected result>` in the project's UI language (`policy.locale`) — the part after `:` becomes "Expected result" in the document.
- **Steps** = `test.step("…")` in the same language, as a tester would do them, using the visible UI names in «».
- **Tags**: `@<entity>` (the DB table name, so the document maps tests to entities), `@smoke` for the critical path (runs on every PR), `@regression` for bug-fix tests.
- **Annotations**: `relation` (`"<from> → <to> (<1:N | flow>)"`) for every test that exercises a relationship; `bug` (`"<ticket>: <what was wrong>"`) for regression tests; optional `id` (`"TC-ORD-001"`) when QA needs fixed ids.
- **Assertions carry a human message** in the UI language: `expect(x, "сума має дорівнювати Σ кількість × ціна").toBe(…)` — it appears as the failure reason in the document.
- Locators by role/label/visible text (`getByRole`, `getByLabel`, `getByText`); `data-testid` only where there is no accessible name. No CSS-class selectors, no `waitForTimeout` — wait for a visible state or a response.
- Tests are independent (own data, any order, parallel-safe).

## What to cover for each entity / feature

From the project's policy and the pattern skills — test the behaviour users rely on, not implementation details:

| Area | Tests |
|------|-------|
| List (`crud-list`) | opens with rows; search by two words in any order finds the created record; new record appears; archived hidden by default and shown (at the end) with "Show archived" |
| Create (`crud-form`) | required-field validation messages; create → lands per `forms.afterCreate` + success toast; **double click on Create → exactly one record** |
| Edit | change a field → Save → stays per `forms.afterSave`, toast, value persists after reload |
| Archive / delete | confirm dialog; archived record disappears from new selections in other forms but still opens |
| Relationships | every FK the UI exposes: pick the parent in the child's form, see the child from the parent's card; "create X from Y" flows from `relations.yaml` incl. their limits (cannot exceed what's left) |
| Documents with lines (`document-lines`) | totals = formula; reopening a saved document doesn't change its numbers; removing a line updates totals |
| Bulk actions (`bulk-actions`) | confirmation shows the server count; result toast; only the selected scope changed |
| Permissions | a non-admin can't do the admin-only action (UI hidden/disabled and the server refuses) |

Keep `@smoke` small (one create + one edit per core entity, ~2–4 min total).

## Regression tests for bug fixes (always)

Every bug fix comes with a regression test, **written first**:

0. Pick the level: the database decides the wrong result (RPC, trigger, constraint, RLS, totals) → **DB test** (`e2e/db/*.db.spec.ts`, rollback fixture, tag `@db`); the bug is in the UI flow → UI test. A DB bug with a visible symptom can have both. Tag `@prod-safe` when it satisfies the rules of *Bug-fix verification on production* — DB tests on the rollback fixture almost always do; then it can also prove a production hotfix.
1. Reproduce the bug as a test (it fails on the current code — run it and see the failure, when the environment allows).
2. Fix the code; the test passes.
3. Title describes the **correct** behaviour ("Пошук «/» не друкує символ у поле: курсор у пошуку, поле порожнє"); tag `@regression` + the entity tag; annotation `bug` = `"<ticket>: <what was wrong>"`.
4. Put it in the module's spec file (not a separate "bugs" file) so it lives next to related cases.

A pure data/SQL bug is no longer an excuse to skip: it is a DB test. Only a bug in an external system or a one-off data repair without a rule behind it may go without a test — say so in the report.

Whatever the level and environment, the run is done only when the residue check passed: zero test rows left.

## `e2e/relations.yaml` (for the testers' document)

```yaml
entities:                      # DB table → label in the UI language, module
  orders: { label: "Замовлення покупців", module: "Продажі" }
  shipments: { label: "Відвантаження", module: "Склад" }
flows:                         # what is created from what — the business view of relationships
  - from: orders
    to: shipments
    action: "Створити відвантаження із замовлення"
    rule: "Не більше, ніж замовлено − вже відвантажено"
```

Add an entity/flow whenever you add tests for it; the document marks entities/flows without tests as "not covered".

## CI (GitHub Actions, fits the free plan)

```yaml
# .github/workflows/e2e.yml
name: e2e
on:
  pull_request: { paths-ignore: ["**/*.md", "docs/**"] }
  schedule: [{ cron: "0 2 * * *" }]   # full suite nightly
  workflow_dispatch:
concurrency: { group: e2e-${{ github.ref }}, cancel-in-progress: true }
jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    env:
      E2E_BASE_URL: ${{ secrets.E2E_BASE_URL }}            # test deployment, or omit to use webServer (npm run dev)
      E2E_EMAIL: ${{ secrets.E2E_EMAIL }}
      E2E_PASSWORD: ${{ secrets.E2E_PASSWORD }}
      E2E_FORBIDDEN_HOSTS: ${{ vars.E2E_FORBIDDEN_HOSTS }}
      NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.TEST_SUPABASE_URL }}
      NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY: ${{ secrets.TEST_SUPABASE_PUBLISHABLE_KEY }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }                            # tags for "what's new" in the testers' document
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: npm }
      - run: npm ci
      - uses: actions/cache@v4
        with: { path: ~/.cache/ms-playwright, key: pw-${{ hashFiles('package-lock.json') }} }
      - run: npx playwright install --with-deps chromium
      - run: npx playwright test ${{ github.event_name == 'pull_request' && '--grep @smoke' || '' }}
      - if: always()
        uses: actions/upload-artifact@v4
        with: { name: e2e-report, path: "playwright-report/\ntest-results/results.json", retention-days: 14 }
```

The testers' PDF can be built in the same job (see `test-docs`). Only Linux runners (macOS minutes cost 10×).

## Audit (reviewer)

| Check | Severity |
|-------|----------|
| a new entity/feature merged without e2e tests (create + edit at least) | WARN (BLOCKER when `policy.e2e.required`) |
| a bug fix without a `@regression` test (or a stated reason it can't be e2e) | BLOCKER |
| test titles/steps not in the UI language, or a title without `:` expected result | WARN |
| missing `@<entity>` tag / `relation` annotation on a relationship test | WARN |
| CSS-class selectors, `waitForTimeout`, tests depending on other tests' data | WARN |
| production URL/keys reachable from the e2e config, credentials in the repo | BLOCKER |
| created records not prefixed/cleaned up, or only archived instead of deleted | WARN (BLOCKER for `@prod-safe`) |
| no `globalTeardown` sweep + residue check (`e2e_cleanup` / `e2e_residue`) once the suite creates data | WARN |
| a table the tests insert into (incl. trigger-made rows) without the `e2e_track` trigger | WARN (BLOCKER when a `@prod-safe` test writes to it) |
| `@prod-safe` test that changes a pre-existing record, calls an unstubbed outward integration, or has no `cleanup.md` entry | BLOCKER |
| DB test not using the rollback fixture (own `begin`/`commit`, service/owner credentials in `E2E_DB_URL`) | BLOCKER |
| the cleanup bypass of an archive-only/append-only guard not tied to `is_e2e_account()` | BLOCKER |
