---
name: rdd-tests
description: Use this skill after rdd-spec to design a characterization test plan for one module that locks observable behavior before any code is rewritten. Triggers on "plan tests for X", "how should I test X", "what tests should I write before refactoring X". Produces rdd/{module}/TESTS.md.
---

# rdd-tests — Plan characterization tests

You are designing the test suite that will **lock the legacy module's observable behavior** so it can be safely rewritten. The tests run against the legacy first (proving they capture current behavior), then against the new implementation (proving parity).

This skill produces a **plan**, not the tests themselves. The plan is reviewed by the user before `/rdd-port` implements it.

## Procedure

### 1. Load configuration and prior artifacts

Read:

- `.rdd.yml` — for `target.test_framework`, `target.test_strategy`, and the legacy source path.
- `{artifacts_dir}/MAP.md`
- `{artifacts_dir}/{module}/SPEC.md` — **required**. If missing, instruct the user to run `/rdd-spec {module}` first.

### 2. Map every business rule to one or more tests

For each `BR-NN` in the spec, design at least one test that **fails if the rule is violated**. Track this as a 1:1 mapping table — coverage gaps are visible.

For each test, record:

- **ID** — `T-NN`, sequentially
- **Covers** — which BRs (one BR can have multiple tests; one test can cover multiple BRs)
- **Type** — integration, unit, property-based, contract
- **Setup** — fixtures needed (DB rows, fake JWT, mock responses)
- **Action** — request/call performed
- **Assertion** — observable outcome checked
- **Why it earns its keep** — one sentence on what regression it prevents

### 3. Identify property-based candidates

Calculations and transformations are often better tested with property-based testing:

- Money calculations (totals, discounts, taxes, commissions)
- Date/time arithmetic (scheduling, billing periods)
- Sorting / pagination invariants
- Round-trip serialization

For each candidate, write the **invariant** in plain language: "for any list of items, `total(items) == sum(item.price * item.qty)`" — this becomes the property in code.

### 4. Identify boundary and edge cases from the legacy code

Re-read the legacy code for branches that won't appear in the spec but matter for parity:

- Empty inputs (empty list, null, missing field)
- Boundary values (0, -1, max int, very long strings)
- Concurrency (two writes at once)
- Retry / idempotency paths
- Timezone-sensitive logic
- Encoding-sensitive logic (unicode, accents — common for non-English systems)

Each boundary case becomes one test (or a property in a property-based test).

### 5. Identify side-effect tests

For each side effect listed in the spec (webhooks, emails, queue messages, audit logs):

- One test per side effect verifying it fires when expected
- One test verifying it does **not** fire when it shouldn't (false-positive guard)
- Idempotency test: replaying the same trigger twice doesn't double-fire

### 6. Apply the anti-babacas filter

For each candidate test, ask: **if this test breaks during a refactor that doesn't change observable behavior, would the user care?**

- **Yes, the user cares** → keep
- **No, it's testing implementation** → drop

Specifically drop:

- Tests that assert internal call sequences ("controller called service.foo")
- Snapshot tests of objects with mutable fields (timestamps, generated ids)
- Tests that re-validate what the validation library already enforces
- Tests of framework internals (DI wiring, decorator application)
- `expect(x).toBeDefined()` without a meaningful assertion

When you drop, note it in a "Rejected tests" section with the reason — so the user can challenge if they disagree.

### 7. Plan the harness

Briefly describe what infrastructure the tests need, based on `target.test_strategy`:

- **Database** — testcontainers? in-memory? real Supabase test project?
- **External APIs** — record/replay? mock at HTTP boundary? sandbox accounts?
- **Auth** — how to mint test JWTs that the legacy and new system both accept
- **Time** — fixed clock? frozen now?
- **Random** — seeded?

This is a planning level — implementation is in `/rdd-port`.

### 8. Plan the dual-target setup

The same test code must run against legacy and new. Patterns:

- `TEST_TARGET=legacy|new` env var that flips the base URL or import
- A test "client" abstraction that both implementations satisfy
- Contract tests written against an OpenAPI/JSON-Schema if one exists

Pick one and document it.

### 9. Write `TESTS.md`

Create `{artifacts_dir}/{module}/TESTS.md` using `templates/TESTS.md`. Sections:

- **Coverage matrix** — table mapping BRs to tests
- **Test list** — one entry per test (ID, covers, type, setup, action, assertion, justification)
- **Property-based tests** — separate section with invariants in plain language
- **Boundary cases** — list with code citations
- **Side-effect tests** — table
- **Rejected tests** — what you considered and dropped, with reasons
- **Harness plan** — DB, mocks, auth, time, random
- **Dual-target plan** — how the same suite runs against legacy and new

### 10. Hand-off

Tell the user:

- Path to `TESTS.md`
- Test count (and rejected count)
- BR coverage: are any BRs unmapped? Flag them.
- Next step: `/rdd-port {module}`

## Quality bar

A good characterization test:

- ✅ Asserts **what** the system does, not **how**
- ✅ Survives reasonable internal refactors
- ✅ Cites the BR(s) it covers in its name or description
- ✅ Fails with a clear, actionable message

A bad characterization test:

- ❌ Couples to internal class names, method signatures, or framework decorators
- ❌ Snapshots whole objects including timestamps, generated ids, or floats with full precision
- ❌ Has no clear connection to a business rule or side effect

## Anti-patterns to avoid

- **Don't aim for line coverage.** Aim for behavior coverage. 60% line coverage with all BRs covered beats 95% with implementation coupling.
- **Don't write the tests yet.** This skill plans; `/rdd-port` implements. The user reviews the plan first.
- **Don't skip rejected tests.** Documenting what you said no to (and why) is a feature, not waste — it teaches the user the heuristic.
