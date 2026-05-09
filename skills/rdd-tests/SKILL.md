---
name: rdd-tests
description: Use this skill after rdd-spec to design a characterization test plan for one module that locks observable behavior before any code is rewritten. Triggers on "plan tests for X", "how should I test X", "what tests should I write before refactoring X". Produces rdd/{module}/TESTS.md.
---

# rdd-tests — Plan characterization tests

You are designing the test suite that will **lock the legacy module's observable behavior** so it can be safely rewritten. Tests run against the legacy first (proving they capture current behavior), then against the new implementation (proving parity).

This skill produces a **plan**, not the tests themselves. The plan is reviewed by the user before `/rdd-port` implements it.

## Procedure

### 1. Load configuration and prior artifacts

Read:

- `.rdd.yml` — for `target.test_framework`, `target.test_strategy`, and the legacy source path
- `{artifacts_dir}/TARGET.md` — for the test strategy posture (integration-first vs unit-first; mocks-where) and error format
- `{artifacts_dir}/MAP.md`
- `{artifacts_dir}/{module}/SPEC.md` — **required**. If missing, instruct the user to run `/rdd-spec {module}` first.

### 2. Validate inputs (pre-flight)

Before designing tests:

- `SPEC.md` Business Rules section is non-empty (at least one BR)
- Every BR has a source citation `(code: ...)`, `(user-confirmed)`, or `(legacy bug, intentional parity)`
- Error Catalog codes referenced by BRs exist in the catalog (no dangling code names)
- Open questions in `SPEC.md` are resolved (or explicitly marked as out-of-scope for this test plan)

If issues found, stop and surface to the user before designing.

### 3. Map every business rule to one or more tests

For each `BR-NN`, design at least one test that **fails if the rule is violated**. Track as a 1:1 coverage matrix — gaps are visible.

Each test record:

- **ID** — `T-NN`, sequential
- **Covers** — which BRs (one BR can have multiple tests; one test can cover multiple BRs)
- **Type** — integration, unit, property-based, contract
- **Setup** — fixtures (DB rows, fake JWT, mock responses)
- **Action** — request/call performed
- **Assertion** — observable outcome checked
- **Why it earns its keep** — one sentence on what regression it prevents

### 4. Write each assertion using AC template formulas

Acceptance criteria — and tests that verify them — follow fixed sentence patterns. These force the test to assert externally-observable behavior, not implementation.

**HTTP endpoint behavior** (most common):

```
[METHOD] [/path] with [input description] returns [HTTP status] with [expected body or error code]
```

Example: `POST /auth/register with an already-registered email returns 409 with EMAIL_ALREADY_EXISTS`

For 204 No Content (action endpoints with no body):

```
[METHOD] [/path] with [input] returns 204 with no response body — [observable side effect]
```

Example: `POST /auth/logout with a valid access token returns 204 with no response body — all user's refresh tokens are revoked`

**Database/persistence behavior** (constraint enforcement, atomicity, integrity):

```
[operation description] — [expected persistence outcome or constraint violation]
```

Example: `Creating a user with channel is atomic — if channel creation fails, no user row is persisted`

**Side-effect behavior** (email sent, event published, job enqueued):

```
[trigger action] causes [observable side effect] containing [key payload element]
```

Example: `Registering a new user causes a confirmation email to be delivered containing a confirmation link with a token`

**Security behavior** (data not leaked, timing-safe, tokens invalidated):

```
[action that probes for information] returns [response that reveals nothing or enforces the boundary]
```

Example: `POST /auth/login with a non-existent email returns 401 with INVALID_CREDENTIALS — same error as wrong password, not revealing email existence`

### 5. BAD vs GOOD test descriptions (calibration examples)

**BAD:** `User registration works and creates an account`
**GOOD:** `POST /auth/register with valid email and password returns 201 with { id, email, name }; a channel is automatically created with handle derived from the email prefix`
**Why:** Specifies method, path, input, status, response shape, and observable side effect — verifiable without reading code.

**BAD:** `Login fails when credentials are wrong`
**GOOD:** `POST /auth/login with a non-existent email returns 401 with INVALID_CREDENTIALS — same error code and status as wrong password, so email existence is not revealed`
**Why:** Makes the security requirement explicit and verifiable.

**BAD:** `Confirmation email is sent after registration`
**GOOD:** `Registering a new user causes a confirmation email to be delivered to the registered address, containing the user's name and a confirmation link with the token`
**Why:** Specifies observable outcome, recipient, key content.

**BAD:** `Refresh token rotation handles theft`
**GOOD:** `POST /auth/refresh with an already-used refresh token returns 401 with TOKEN_REUSE_DETECTED and all refresh tokens in the same rotation family are revoked`
**Why:** Describes exact trigger, response, and persistence consequence.

### 6. Boundary rules — what goes in tests vs elsewhere

| Concern | Where it goes | Example |
|---------|---------------|---------|
| What the system does externally | **Test assertion** | `POST /auth/login with valid credentials returns 200 with { access_token, refresh_token }` |
| How to build it internally | **`/rdd-port` technical actions** (not in TESTS.md) | `Hash password using argon2.hash()` |
| Implementation choices (lib, ORM, framework) | **`TARGET.md` (TDs)** or `/rdd-port` | `Validate with zod via nestjs-zod` |
| Which test files exercise the AC | **TESTS.md "Test list" + file column** | `auth.e2e-spec.ts` |

If a test asserts how the system does something internally (calls service.foo, uses ORM method bar, has DI provider X), it belongs nowhere — delete it.

### 7. Identify property-based candidates

Calculations and transformations are often better tested with property-based testing:

- Money calculations (totals, discounts, taxes, commissions)
- Date/time arithmetic (scheduling, billing periods)
- Sorting / pagination invariants
- Round-trip serialization

For each candidate, write the **invariant** in plain language: *"for any list of items and any discount in [0, total]: `total = max(0, sum(items) - discount)` rounded half-up"* — this becomes the property in code.

### 8. Identify boundary and edge cases from legacy code

Re-read legacy code for branches that won't appear in the spec but matter for parity:

- Empty inputs (empty list, null, missing field)
- Boundary values (0, -1, max int, very long strings)
- Concurrency (two writes at once)
- Retry / idempotency paths
- Timezone-sensitive logic
- Encoding-sensitive logic (unicode, accents — common for non-English systems)

Each boundary case becomes one test (or a property in a property-based test).

### 9. Identify side-effect tests

For each side effect listed in `SPEC.md`:

- One test per side effect verifying it fires when expected
- One test verifying it does **not** fire when it shouldn't (false-positive guard)
- Idempotency test: replaying the same trigger twice doesn't double-fire

### 10. Apply the anti-babacas filter

For each candidate test, ask: **if this test breaks during a refactor that doesn't change observable behavior, would the user care?**

- **Yes, the user cares** → keep
- **No, it's testing implementation** → drop

Specifically drop:

- Tests that assert internal call sequences ("controller called service.foo")
- Snapshot tests of objects with mutable fields (timestamps, generated ids)
- Tests that re-validate what the validation library already enforces
- Tests of framework internals (DI wiring, decorator application)
- `expect(x).toBeDefined()` without a meaningful assertion

When you drop, note in **Rejected tests** with the reason — so the user can challenge.

### 11. Validation checklist (apply per test before finalizing)

For each test in your draft, verify:

1. **Observable?** Can it be verified by calling an endpoint, checking a mailbox, querying a DB — without reading source code?
2. **Specific?** Names HTTP method, path, status code, error code, or observable outcome?
3. **Scoped?** Belongs to exactly this module's BRs, not another module's?
4. **Non-redundant?** Not already covered by another test in this or a different module?
5. **Behavioral, not implementation?** Describes what the system does — not how internally?
6. **Distinct from `TARGET.md` decisions?** Doesn't assert lib choice, ORM, framework — those are decided elsewhere.

If any answer is no, fix or drop the test.

### 12. Plan the harness and dual-target setup

**Harness:**

- **Database** — testcontainers? in-memory? real Supabase test project?
- **External APIs** — record/replay? mock at HTTP boundary? sandbox accounts?
- **Auth** — how to mint test JWTs that legacy and new system both accept
- **Time** — fixed clock? frozen now?
- **Random** — seeded?

**Dual-target:**

- `TEST_TARGET=legacy|new` env var that flips base URL or import
- A test "client" abstraction satisfied by both implementations
- Contract tests written against an OpenAPI/JSON Schema if one exists

Pick one and document.

### 13. Write `TESTS.md`

Use `templates/TESTS.md`. Sections:

- **Coverage matrix** — table mapping BRs to tests
- **Test list** — one row per test (ID, covers, type, setup, action, assertion, justification)
- **Property-based tests** — separate section with invariants in plain language
- **Boundary cases** — list with code citations
- **Side-effect tests** — table
- **Rejected tests** — what you considered and dropped, with reasons
- **Harness plan** — DB, mocks, auth, time, random
- **Dual-target plan** — how the same suite runs against legacy and new

### 14. Hand-off

Tell the user:

- Path to `TESTS.md`
- Test count (and rejected count)
- BR coverage: are any BRs unmapped? Flag them.
- Next step: `/rdd-port {module}`

## How many tests per BR

- **Minimum: 1** — happy path AC
- **Typical: 1–3** — happy path + error branch(es) + side-effect verification
- **Soft cap: 5 per BR** — if a BR needs more, the BR is too big and should be split

## Anti-patterns to avoid

- **Don't aim for line coverage.** Aim for behavior coverage. 60% line coverage with all BRs covered beats 95% with implementation coupling.
- **Don't write the tests yet.** This skill plans; `/rdd-port` implements. The user reviews the plan first.
- **Don't skip rejected tests.** Documenting what you said no to (and why) is a feature — it teaches the user the heuristic.
- **Don't reuse generic "should work" descriptions.** Use the AC template formulas.
- **Don't test what `TARGET.md` decided.** Lib choice and framework are not behavior.
