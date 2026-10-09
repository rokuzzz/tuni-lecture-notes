# COMP.SE.200 — Software Testing

## Introduction to testing

Testing does not improve quality — it produces information about quality.
Improvement only happens when someone acts on that information. A green CI run
means "the checks we wrote passed", not "it works".

**Mindset when writing a test.** A successful run is one that causes a failure.
Write a test to confirm the code works and you'll get the happy path. Write it to
break the code and you start asking about empty arrays, null, a 500 from an
upstream service, race conditions. Testing your own code replays your own faulty
mental model — that's the point of review and cross-testing.

**Oracle** — the source the expected result comes from, plus the mechanism that
delivers the verdict. `toBe(150)` is an imprint of the oracle, not the oracle
itself. Two identical tests can mean opposite things: 150 derived from the spec
(verifies a requirement) or copied from the code's output (locks in current
behaviour, bugs included). Before writing a test, ask: how do I know the correct
answer?
- Sources: specification, previous version, reference implementation, domain expert.
  No oracle is perfect.
- Snapshot tests and "oracle = the old version" cement existing bugs.

**Oracle problem** — cases where the correct output is unknown (ranking, ML,
generation, performance). Ways around it:
- invariants instead of values: result is sorted, sum is preserved, idempotency
- metamorphic relations: reordering the input doesn't change the answer, doubling the input doubles the result
- bounds instead of a point: latency < 200ms
- reference implementation: a slow known-correct version checks the fast one

**What you cannot demand from testing:** finding all bugs, assuring quality (tests
measure, they don't assure), setting the release date (a business decision). What
you can demand: information about the product and clear communication of problems —
finding a bug is half the work, the other half is convincing someone it's worth fixing.

---

## Levels of testing

Four levels by scope, narrow to broad: **unit → integration → system → acceptance**.
The broader the scope, the later the feedback and the worse the localisation:
"something in checkout broke" vs "line 42 in calculateDiscount".

Practical question is not "how many tests do we have" but **at which level each
thing is checked**. Testing business logic through E2E works but is slow, flaky
and diagnoses poorly. Don't check at the top what can be checked at the bottom —
that's the whole point of the test pyramid, and the pyramid shape is just a
consequence of cost, not a rule. If your integration tests are fast and stable
(testcontainers), their share can be larger.

**Trade-off:** lower levels give faster feedback but couple tests to code structure —
unit tests break on refactoring. Higher levels know nothing about implementation
and survive a rewrite.

**System vs acceptance** — technically the same E2E scenario, but a different
question and a different oracle. System: does it match the spec (verification).
Acceptance: does it solve the user's problem (validation). They can diverge — the
system matches the spec perfectly and the spec described the wrong thing.

Levels are about scope, not chronology — all four can happen in one sprint.

---

## Test doubles

A unit rarely runs alone — it has a caller above and dependencies below. To test it
in isolation, both sides get replaced.

- **Driver** — calls the unit, feeds data, collects results. In practice this is the
  test runner: `it(...)` is the driver.
- **Stub** — a dumb replacement for a dependency, returns a fixed value.
- **Mock** — same position, but it *records calls*, so you can assert on how the unit
  talked to it. `jest.fn()` is a plain function plus a log of arguments.

**The distinction that matters is what the assert points at** — the returned value,
or the dependency itself.

```js
async function describeUser(id, db) {
  const name  = (await db.findUser(id)).name;
  const email = (await db.findUser(id)).email;   // two round trips
  return `${name} <${email}>`;
}

// A — assert on the result
const db = { findUser: async () => ({ name: "roman", email: "a@b.c" }) };
expect(await describeUser(1, db)).toBe("roman <a@b.c>");

// B — assert on the calls
const db = { findUser: jest.fn(async () => ({ name: "roman", email: "a@b.c" })) };
await describeUser(1, db);
expect(db.findUser).toHaveBeenCalledTimes(2);
```

Now optimise to a single query — same interface, same output, strictly better code.
A passes. B fails with `expected 2, received 1`. The test broke because it was
pinned to call count, not to behaviour. Same thing happens when you add a cache or
batch requests.

(Renaming `findUser` breaks both tests — that's a real contract change and should
break them. The danger of call assertions is breaking on changes that are invisible
from the outside.)

**Rule:** assert on calls only when the call *is* the contract — email sent, event
published, payment charged. Such functions return nothing; the side effect is the
whole point. Working test: *if this call disappeared, would anyone outside notice?*
Email not sent — yes, that's the contract. Two DB queries collapsed into one — no,
that's internal plumbing.

```js
expect(mailer.send).toHaveBeenCalledWith("a@b.c", "Payment failed");  // ok — what it promises
expect(db.findUser).toHaveBeenCalledTimes(1);                         // bad — how it's built
```

**Main value of doubles: negative testing.** Error branches are the hardest to reach
and the most likely to break in production. Triggering a real 500, a timeout, an
expired token or a duplicate key is expensive or impossible; a stub that throws is
three lines:

```js
const api = { fetchUser: async () => { throw { status: 500 }; } };
expect(await getProfile(1, api)).toBe(cachedValue);
```

Without doubles, `catch` blocks typically never execute in any test — which is how a
typo in error handling silently swallows failures in production.

---

## TDD

TDD is a **design method more than a testing method** — the main part is defining
behaviour, not finding bugs. Its real payoff: you design the interface before the
implementation and immediately feel whether it's usable. An API that's awkward in a
test will be awkward in production.

**Weak spot: negative testing.** TDD mechanically pulls toward the happy path — you
write a test for what should work, then make it work. Error branches, boundaries and
malformed input don't show up on their own, because a test written before the code
describes *intent*, and bugs live in what you didn't think of. Full TDD coverage and
weak testing coexist comfortably.

Practical consequence: after the red-green-refactor cycle, do a separate "how do I
break this" pass.

No evidence TDD produces better unit tests than other practices; over-systematising
can hurt, since testing also needs unplanned, exploratory thinking.
