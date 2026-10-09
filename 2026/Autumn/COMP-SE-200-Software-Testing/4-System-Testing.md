# COMP.SE.200 — Software Testing

## System testing

System testing checks a **deployed build** of the whole system from the outside. Lower levels
test **code** (functions, modules, their logic); system-level tests test a **build** —
the deployed artifact in a concrete environment. That's why they catch what unit tests
can't: missing env variable, wrong config, unapplied migration, a service that can't
reach the DB or Kafka. The code is perfect, every unit test is green, and the deployed
system is dead.

**Build** — the runnable result of turning source code into something you can deploy:
compile (`tsc` → `dist/`), pull dependencies, package (often a Docker image with a tag).
Code can be correct and the build broken (wrong library version, missing config file,
built for the wrong env). Every build has an identifier (tag / commit hash) — "on which
build?" is the first question about any bug on staging.

**Environments**, roughly in order:
- **local** — my machine, unit tests, manual poking
- **PR / preview environment** — temporary environment built from one PR commit;
  E2E / acceptance tests run there before merge
- **staging** — prod-like environment without real users, usually after merge
- **production** — the *same* build that was verified, not a rebuild

### Smoke test

A smoke test doesn't look for bugs — it decides **whether further testing makes sense**.
Name comes from electronics: power on the board; if it smokes, stop. A broken build
under a full E2E run produces a hundred red tests that say nothing except "the build
is dead", and someone wastes an hour reading them.

Properties: broad not deep, fast, automated (runs on every deploy), fails for obvious
reasons, runs in the same environment as the tests that follow. Manually poking the
feature before pushing is a smoke test too — the definition is about *why*, not
*who* or *when*.

System-level tests **don't import the code**. They act as an external client and need
only the address of the running system:

```ts
// smoke.test.ts
const BASE_URL = process.env.BASE_URL; // e.g. https://staging.myapp.com

test("service is up", async () => {
  const res = await fetch(`${BASE_URL}/health`);
  expect(res.status).toBe(200);
});

test("login works", async () => {
  const res = await fetch(`${BASE_URL}/api/login`, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ email: "smoke@test.com", password: process.env.SMOKE_PASSWORD }),
  });
  expect(res.status).toBe(200);
});
```

Run as a separate pipeline step right after deploy:

```bash
BASE_URL=https://staging.myapp.com npm run test:smoke
```

Red stops the pipeline (and rolls back), green lets the expensive tests run.

### System / E2E tests

System / E2E tests go deeper — a whole scenario end to end, through a real browser
(Playwright, Cypress) or a chain of API calls:

```ts
test("user can create a campaign", async ({ page }) => {
  await page.goto(`${BASE_URL}/login`);
  await page.fill("#email", "e2e@test.com");
  await page.fill("#password", process.env.E2E_PASSWORD!);
  await page.click("button[type=submit]");
  await page.click("text=New campaign");
  await page.fill("#name", "Test campaign");
  await page.click("text=Save");
  await expect(page.locator("text=Test campaign")).toBeVisible();
});
```

|Unit|System|
|---|---|
|`import { calculate }` → call → check|No code visible → request to a URL → check the response|

**Happy-path-only E2E is mostly fine** — error branches are cheaper to cover lower,
with a stub that throws. But some negatives exist only at system level and should
have at least one E2E:
- permissions (user A can't open user B's data)
- what the user sees when the backend fails
- form validation
- double-submit not creating duplicates

Check: *"if the backend errors at this step, does **any** test at **any** level cover it?"*

### Results belong to one build

A test round on a system that changes mid-round gives no information: half the
scenarios ran on one version, half on another, and a failure can't be attributed.
Verify a specific build (tag / commit), report "verified on `a1b2c3d`", and run smoke/E2E
in the same pipeline run as the deploy. Preview environments per PR solve this by
construction — one commit, isolated, nobody else deploys there.

### Bug reports

A bug nobody fixes is worth the same as a bug nobody found. A report has to sell it:
- reproducible steps
- expected vs actual
- build + environment
- **impact** — decides whether it gets into the sprint

Report fast and directly — the later it reaches the author, the more expensive the fix.

---

## System integration testing

Between systems, what breaks is not code but **agreements**. Each side tests against
its own assumptions about the other. Assumptions match → works. They diverge → both
sides green, the whole broken, and nobody specifically is at fault.

```json
{ "orderId": 42, "amount": 1999, "createdAt": 1760025600 }
```

Producer means cents and seconds, consumer reads euros and milliseconds → an invoice
for 1999 € dated January 1970. Same mechanism as wrong stubs, at a larger scale.

Where assumptions typically diverge:
- **data shape** — parameter order, cents vs euros, s vs ms, float for money, int64
  losing precision in JS, encodings (UTF-8 vs Latin-1), `null` vs missing vs `[]`
- **ordering** — `order.paid` arriving before `order.created` (partitions, retries)
- **what happened at all** — timeouts

### Timeouts and idempotency

**Timeouts** are the worst case. A asks B to charge 50 €, B charges but answers slowly,
A times out and retries, B charges again. Both followed their own logic; they disagree
on whether the operation happened.

Defence: **idempotency** — A sends an `Idempotency-Key`, B returns the previous result
for a repeated key instead of executing again. Test it with a stub that answers late or
drops the connection and then succeeds on retry, and assert the operation ran once.

### What to do about it

1. One shared contract/schema that both sides test against, instead of two private
   assumptions.
2. Test unpleasant variants, not only the intended flow: out-of-order, late, duplicate.
3. Give the integration an owner — otherwise each side is sure its part is fine.
4. Integrate early. It never works on the first attempt.
