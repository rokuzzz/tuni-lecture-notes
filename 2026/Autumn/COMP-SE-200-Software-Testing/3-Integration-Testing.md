# COMP.SE.200 — Software Testing

## Integration testing

Units work in isolation; the question is whether they work together. The level
checks that interfaces are used correctly and that interaction goes as intended —
including whether exceptions thrown by one unit are handled correctly by another.

**Integration ≠ integration testing.** Integration is the fact of assembly — it
compiled, linked, started. Integration testing is checking that the assembled thing
actually works. CI covers the first and creates the illusion of the second.

> "The integration in itself says nothing about quality. The tests must also be good.
> Repeating unit tests on the server is not enough."

**Why the gap exists.** Unit tests run units in isolation with dependencies replaced.
Running them a thousand times on a server never checks that real module A talks to
real module B correctly. The gap sits exactly where the stubs were: a stub reflects
*your assumption* about the dependency — does the real DB return `null`, `undefined`,
an empty array, or throw when the record is missing? Wrong assumption, green unit
test, broken integration.

**Stubs cost more than drivers.** A driver just calls the code and inspects the
result. A stub has to *impersonate* something, which means you must already know how
the real dependency behaves in every case — missing records, exceptions, timeouts,
encodings. The cost isn't the lines of code, it's needing that knowledge to be correct.

**Test tiers.** Feedback has to be fast, so one suite isn't enough: quick tests on
every commit, slow ones (long integration, load, static analysis, architecture checks)
nightly. Cramming everything into one 40-minute pipeline is how developers start
merging without waiting for it.

**Integration order:** connect the riskiest and critical-path pieces first — the
higher the risk, the earlier you need to know. Verify unfamiliar technology in a
separate proof of concept outside the main codebase, before it's load-bearing.
