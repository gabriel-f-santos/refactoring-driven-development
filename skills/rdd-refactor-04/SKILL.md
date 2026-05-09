---
name: rdd-refactor-04
description: Use this skill after rdd-specify-03 to plan characterization tests, lock legacy behavior, and port the module to the target stack with parity. Fourth step in the RDD pipeline (per-module). Combines test planning + implementation + port — TDD and refactoring are inseparable phases of the same act. Triggers on "refactor X", "port X", "rewrite X with parity", "implement the migration of X". Produces TESTS.md, working code, REFACTOR.progress.md.
---

# rdd-refactor — Plan tests, lock legacy, port with parity

You are executing the rewrite of one module. This is the only RDD skill that produces runnable code. The skill merges what would otherwise be a redundant handoff between "plan tests" and "implement port" — both are part of the same refactoring act.

You must:

1. **Plan characterization tests** from `SPEC.md` (output `TESTS.md`, user reviews — STOP)
2. **Set up the test harness**
3. **Implement the tests against the legacy** until they pass — proves they capture current behavior
4. **Scaffold the new module** in the target stack
5. **Port the module** entry point by entry point with parity (parity-first, not idiomatic-first)
6. **Verify side effects**
7. **Run final verification** (full suite + type-check + build)
8. **Stop.** Idiomatic improvement is `/rdd-improve-05` — a separate session.

The single most common failure mode is rushing through entry points without verifying parity at each step. Treat the per-entry-point STOP as a hard terminator.

---

## Modes

- **Default (pause)**: stop after each entry point port, emit a completion line, ask `Continuar para <next>?`, and **wait** for the user's reply in a new turn. Also stop after Phase 1 for the user to review `TESTS.md` before any code is implemented.
- **Continuous (autopilot)**: only when the user **explicitly** requested it at session start with phrases like "execute tudo", "autopilot", "don't pause between entry points", "run all at once". When unsure, default to pause.

---

## Inputs

The user invokes the skill with a module name (e.g., "/rdd-refactor-04 products"). Resolve:

- `{artifacts_dir}/TARGET.md` — must exist (chosen test framework, test posture, error format, conventions)
- `{artifacts_dir}/MAP.md` — must exist
- `{artifacts_dir}/{module}/SPEC.md` — must exist with BRs and Error Catalog filled

If any is missing, instruct the user to run the upstream skill first. **Do not proceed.**

`TESTS.md` is **produced by this skill** in Phase 1 — don't expect it as input.

---

## Progress file

`{artifacts_dir}/{module}/REFACTOR.progress.md`:

```markdown
# {module} — Refactor Progress

**Status:** in_progress | completed
**Phase:** preflight | plan_tests | harness | lock_legacy | scaffold | port_entrypoints | side_effects | final_verification | done
**Entry points:** X/Y completed

## Phases

### preflight
- **Status:** completed | pending
- **Notes:** ...

### plan_tests
- **Status:** completed | pending
- **TESTS.md generated:** rdd/{module}/TESTS.md
- **Tests planned:** N (rejected: M)
- **Coverage:** all BRs mapped | gaps: BR-XX, BR-YY

### harness
- **Status:** completed | pending
- **Notes:** ...

### lock_legacy
- **Status:** completed | pending
- **Tests passing on legacy:** X/Y

### scaffold
- **Status:** completed | pending

### port_entrypoints

#### EP-1: <name>
- **Status:** completed | pending
- **Tests on new:** X/Y passing
- **Tests on legacy:** X/Y passing (parity check)
- **Observations:** out-of-scope notes or "none"

(repeat per entry point)

### side_effects
- **Status:** completed | pending

### final_verification
- **Status:** completed | pending
- **Full suite (new):** pass | fail
- **Full suite (legacy):** pass | fail
- **Type check:** pass | fail
- **Build:** pass | fail
```

This is the source of truth for resuming and for the final completion report. `/rdd-status` reads it.

---

## Procedure

### Phase 0 — Preflight

Before designing tests:

1. **Inputs present.** `TARGET.md`, `MAP.md`, `{module}/SPEC.md` exist. `SPEC.md` Business Rules section is non-empty; every BR has a source citation; Error Catalog codes referenced by BRs exist in the catalog. Open Questions in `SPEC.md` are resolved.
2. **Branch check.** `git status` and `git branch --show-current`. If on `main` or default, or uncommitted changes touching files outside the migration scope, **stop** and ask the user to set up the right branch.
3. **Target subproject readiness.** `target.source` directory exists, dependencies installed.
4. **Resume check.** Look for `REFACTOR.progress.md`. If found, read it: which phase to resume, which entry points completed. Tell the user: *"Found progress file: phase=`port_entrypoints`, EP X/Y completed. Resuming from EP-N."*
5. **Mode check.** Continuous mode only if explicitly requested at session start. Default = pause.

If preflight passes, create or update `REFACTOR.progress.md`. Create the persistent task list (`TaskCreate`):

- Plan tests
- Set up harness
- Lock legacy
- Scaffold
- Port EP-1: <name>
- Port EP-2: <name>
- ...
- Verify side effects
- Final verification

Mark each task `in_progress` immediately before its work; `completed` at the boundary defined for that phase.

### Phase 1 — Plan characterization tests

Design the test suite that will lock the module's observable behavior. Tests run against legacy first (proving they capture current behavior), then against the new implementation (proving parity).

#### 1.1 — Map every business rule to one or more tests

For each `BR-NN` in `SPEC.md`, design at least one test that **fails if the rule is violated**. Track as a 1:1 coverage matrix — gaps are visible.

Each test record: `T-NN`, covers (BRs), type (integration/unit/property/contract), setup, action, assertion, why it earns its keep.

#### 1.2 — Write each assertion using AC template formulas

Acceptance criteria — and the tests that verify them — follow fixed sentence patterns. These force assertions on externally-observable behavior, not implementation.

**HTTP endpoint behavior:**
```
[METHOD] [/path] with [input description] returns [HTTP status] with [expected body or error code]
```
Example: `POST /auth/register with an already-registered email returns 409 with EMAIL_ALREADY_EXISTS`

**For 204 No Content:**
```
[METHOD] [/path] with [input] returns 204 with no response body — [observable side effect]
```
Example: `POST /auth/logout with a valid access token returns 204 with no response body — all user's refresh tokens are revoked`

**Database/persistence behavior:**
```
[operation description] — [expected persistence outcome or constraint violation]
```
Example: `Creating a user with channel is atomic — if channel creation fails, no user row is persisted`

**Side-effect behavior:**
```
[trigger action] causes [observable side effect] containing [key payload element]
```
Example: `Registering a new user causes a confirmation email to be delivered containing a confirmation link with a token`

**Security behavior:**
```
[action that probes for information] returns [response that reveals nothing or enforces the boundary]
```
Example: `POST /auth/login with a non-existent email returns 401 with INVALID_CREDENTIALS — same error as wrong password, not revealing email existence`

#### 1.3 — BAD vs GOOD calibration examples

**BAD:** `User registration works and creates an account`
**GOOD:** `POST /auth/register with valid email and password returns 201 with { id, email, name }; a channel is automatically created with handle derived from the email prefix`

**BAD:** `Login fails when credentials are wrong`
**GOOD:** `POST /auth/login with a non-existent email returns 401 with INVALID_CREDENTIALS — same error code and status as wrong password, so email existence is not revealed`

**BAD:** `Confirmation email is sent after registration`
**GOOD:** `Registering a new user causes a confirmation email to be delivered to the registered address, containing the user's name and a confirmation link with the token`

#### 1.4 — Boundary rules: what goes in tests vs elsewhere

| Concern | Where it goes | Example |
|---------|---------------|---------|
| What the system does externally | **Test assertion** | `POST /auth/login with valid credentials returns 200 with { access_token, refresh_token }` |
| How to build it internally | **Phase 5 technical actions** (not in TESTS.md) | `Hash password using argon2.hash()` |
| Implementation choices (lib, ORM, framework) | **TARGET.md (TDs)** or Phase 5 | `Validate with zod via nestjs-zod` |
| Which test files exercise the AC | **TESTS.md "Test list" file column** | `auth.e2e-spec.ts` |

If a test asserts how the system does something internally (calls service.foo, uses ORM method bar, has DI provider X), delete it.

#### 1.5 — Identify property-based candidates

Calculations and transformations are often better tested with property-based testing:

- Money calculations (totals, discounts, taxes, commissions)
- Date/time arithmetic (scheduling, billing periods)
- Sorting / pagination invariants
- Round-trip serialization

For each, write the **invariant** in plain language. Example: *"for any list of items and any discount in [0, total]: `total = max(0, sum(items) - discount)` rounded half-up"*.

#### 1.6 — Identify boundary and edge cases from legacy code

Re-read legacy for branches that won't appear in the spec but matter for parity:

- Empty inputs (empty list, null, missing field)
- Boundary values (0, -1, max int, very long strings)
- Concurrency (two writes at once)
- Retry / idempotency paths
- Timezone-sensitive logic
- Encoding-sensitive logic (unicode, accents)

Each becomes a test (or property in a property-based test).

#### 1.7 — Identify side-effect tests

For each side effect listed in `SPEC.md`:

- One test per side effect verifying it fires when expected
- One test verifying it does **not** fire when it shouldn't (false-positive guard)
- Idempotency test: replaying the same trigger twice doesn't double-fire

#### 1.8 — Apply the anti-babacas filter

For each candidate test, ask: **if this test breaks during a refactor that doesn't change observable behavior, would the user care?** Yes → keep. No → drop. Specifically drop:

- Tests asserting internal call sequences
- Snapshot tests of objects with mutable fields (timestamps, generated ids)
- Tests re-validating what the validation library already enforces
- Tests of framework internals
- `expect(x).toBeDefined()` without a meaningful assertion

When you drop, note in **Rejected tests** with the reason.

#### 1.9 — Validation checklist (apply per test)

For each test, verify: Observable? Specific? Scoped? Non-redundant? Behavioral, not implementation? Distinct from `TARGET.md` decisions? If any answer is no, fix or drop.

#### 1.10 — Plan the harness and dual-target setup

**Harness** per `target.test_strategy`:
- Database (testcontainers? in-memory? real test project?)
- External APIs (record/replay? mock at HTTP boundary? sandbox?)
- Auth (test JWTs both systems accept)
- Time (fixed clock?)
- Random (seeded?)

**Dual-target:**
- `TEST_TARGET=legacy|new` env var flipping base URL or import
- Test client abstraction satisfied by both implementations

#### 1.11 — Write `TESTS.md`

Use `templates/TESTS.md`. Sections: Coverage matrix, Test list, Property-based tests, Boundary cases, Side-effect tests, Rejected tests, Harness plan, Dual-target plan.

Update `REFACTOR.progress.md`: `plan_tests` → `completed`, record counts. **STOP** (default mode) and emit:

```
Tests planned for {module}: N tests covering Y BRs (rejected: M).
Path: rdd/{module}/TESTS.md
Continuar para harness?
```

In continuous mode, skip the question and proceed to Phase 2.

### Phase 2 — Set up the test harness

Per the harness plan from Phase 1:

- Install/configure the test framework (`target.test_framework`)
- Wire up the test database
- Build the dual-target client abstraction
- Configure auth, time, random

Commit this as a discrete, reviewable change.

Update `REFACTOR.progress.md`: `harness` → `completed`. **STOP** (default mode) and ask `Continuar para lock_legacy?`.

### Phase 3 — Lock legacy behavior

Implement every test from `TESTS.md` pointing at the **legacy** implementation. Run them. **Expect them to pass.**

If a test fails, three situations:

- **Test is wrong** → fix the test
- **Spec is wrong** → update `SPEC.md`, then fix the test
- **Legacy has a bug not in spec** → discuss with user; default is to update `SPEC.md` to match legacy (parity), mark BR as `(legacy bug, intentional parity)`

Never adjust legacy code to make tests pass. Legacy is the source of truth at this stage.

When all tests are green against legacy, commit. **This is the behavior lock.**

Update `REFACTOR.progress.md`: `lock_legacy` → `completed`, record `Tests passing on legacy: X/Y`. **STOP** and ask `Continuar para scaffold?`.

### Phase 4 — Scaffold the new module

Create the module under `target.source` per `TARGET.md` conventions:

- Module/package directory
- Entry points matching the spec's API surface (one route per spec endpoint)
- Empty implementations returning 501 or throwing "not implemented"
- Required DI / module registration

Tests against the new target should fail cleanly.

Commit. Update `REFACTOR.progress.md`: `scaffold` → `completed`. **STOP** and ask `Continuar para port_entrypoints?`.

### Phase 5 — Port entry points (the per-EP loop)

For each entry point in order (read from `SPEC.md` API surface), execute these steps. **Do not batch.**

#### Step 5.1 — Plan the EP

Re-read the EP's section in `SPEC.md` and the corresponding tests in `TESTS.md`. Internal checklist: one item per BR + one per side effect + "run tests on new". Mark this EP's task `in_progress`.

#### Step 5.2 — Port the logic (parity-first)

- **Copy the structure** from legacy, adapting only for syntax differences (Deno → Node, JS → TS, framework idioms required to compile)
- **Keep the same control flow.** Same `if` order, same early returns, same error responses. Even if it looks ugly.
- **Keep the same DB queries.** Translate 1:1, not "the way the new ORM prefers"
- **Keep the same field names** in responses
- **Keep bugs that callers depend on** (`legacy bug, intentional parity`)

Stay in scope: only files required by **this EP**. Unrelated issues → record in `REFACTOR.progress.md` under EP `Observations`, do **not** act on them.

#### Step 5.3 — Run only this EP's tests against new

```
TEST_TARGET=new <test-command for this EP's tests>
```

Not the full suite. Save that for final verification.

#### Step 5.4 — Fix loop (max 3 attempts)

If green, proceed to step 5.5.

If red, enter the fix loop. Read failure output, diagnose root cause, apply focused fix, re-run **the same tests**. Maximum **3 attempts**. Count deliberately.

**Discipline:**
- **Read the error.** Same fix applied twice = wrong diagnosis.
- **Fix the root cause.** Don't weaken tests. Don't add `.skip`, `.only`, `xit`. Don't catch and swallow errors.
- **Stay in scope.** Failure pointing to a previously-completed EP → **stop**, escalate.
- **No shortcuts.** Never disable hooks. Never bypass safety checks.

After 3 unsuccessful attempts, **stop**. Report which EP, concise failure output, root-cause hypothesis, attempts made. Wait for user.

#### Step 5.5 — Run the same tests against legacy (parity check)

```
TEST_TARGET=legacy <test-command for this EP's tests>
```

Both must be green. If a test passes on `new` but fails on `legacy`, you accidentally fixed a legacy bug. Decide with the user: revert (parity) or document under `Intentional deviations from legacy` in `SPEC.md`.

#### Step 5.6 — STOP after the EP

Update the EP's task `completed`. Update `REFACTOR.progress.md`: this EP's status → `completed`, record test counts, fold any out-of-scope observations.

These are the **last tool calls** allowed before the stop. Then emit:

```
EP-N (<name>) ported. Tests on new: X/Y passing. Tests on legacy: X/Y passing.
Continuar para EP-N+1?
```

**STOP.** Wait for the user's reply. Do not call further tools. Do not flip the next EP's task.

In continuous mode, skip the question and proceed to step 5.1 of the next EP.

When the last EP is done, skip the question (the per-EP report folds into the final Completion report) and proceed to Phase 6.

### Phase 6 — Verify side effects

Run all side-effect tests against `TEST_TARGET=new`. Pay extra attention:

- **Webhook payloads** — must match byte-for-byte (modulo timestamps, ids)
- **Email/notification content** — must match
- **Audit log entries** — must match shape
- **Queue messages** — must match shape and routing key

If any side effect can't match exactly, document under `Intentional deviations from legacy` in `SPEC.md`.

Update `REFACTOR.progress.md`: `side_effects` → `completed`. Proceed to Phase 7.

### Phase 7 — Final verification

Run:

1. **Full module test suite** against `TEST_TARGET=new`
2. **Full module test suite** against `TEST_TARGET=legacy` (confirm legacy didn't regress while editing the harness)
3. **Type-check** the target subproject
4. **Build** the target subproject

Apply the 3-attempt fix-loop if anything fails. Stop and report if still failing after 3.

When all green, update `REFACTOR.progress.md`: `final_verification` → `completed`, `Status` → `completed`, `Phase` → `done`.

### Phase 8 — Stop. Do not refactor for style yet.

This is the hardest discipline step. The new code is parity-correct but probably not idiomatic. **Stop anyway.**

Idiomatic refactoring is `/rdd-improve-05` — a separate session. The tests from this skill guard that refactor. That's their second job.

---

## Completion report

When the module is fully refactored:

- Tests passing on legacy: X/Y
- Tests passing on new: X/Y
- Type-check, build status
- Any intentional deviations recorded in `SPEC.md`
- Out-of-scope observations aggregated from `REFACTOR.progress.md`

Next steps in the user's hands:
- **Cutover** (feature flag, gradual rollout, monitoring)
- **Idiomatic improvement** via `/rdd-improve-05`
- **Cleanup** (delete legacy code after stable cutover)

Git operations beyond local commits (push, PR) are out of scope — the user owns version control. You commit at the boundaries specified above but do not push.

---

## Anti-patterns to avoid

- **Don't refactor while porting.** "Improving while you're already there" turns a parity port into a behavior change. Phase 8 stop is non-negotiable.
- **Don't translate to "idiomatic" early.** Even if legacy is bad, copy shape-by-shape. Clean up after parity is proved.
- **Don't skip the legacy-side test run** (Phase 3 and step 5.5). Tests never validated against legacy might pass on new for the wrong reason.
- **Don't merge cutover into the refactor.** This skill produces a parity-correct module behind a feature flag. Flipping the flag is separate.
- **Don't trust "looks the same" — run the suite.** Tiny differences (rounding, defaults, sort order) bite in production.
- **Don't violate the STOP.** "Continuar para EP-N+1?" is a terminator, not a rhetorical question.
- **Don't lose count in the fix loop.** 3 attempts means 3.
- **Don't quietly edit a completed EP.** If a failure points back to one, escalate.
- **Don't push to remote.** User owns git operations beyond local commits.

---

## When to spawn sub-agents

For long porting tasks (>1000 lines per EP), spawn the appropriate code-writing agent for the target stack with explicit instructions: "parity-first, copy structure, do not refactor". Pass it the legacy file + spec section. **Verify against the test suite** before accepting. Do not delegate the fix loop — run that yourself with full context.
