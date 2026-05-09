---
name: rdd-port
description: Use this skill after rdd-tests to implement the characterization test suite, run it against the legacy system, then port the module to the target stack with parity. Triggers on "port X", "rewrite X with parity", "implement the migration of X". Produces working code + green tests on both legacy and new.
---

# rdd-port — Implement tests, then port with parity

You are executing the rewrite. This is the only skill in the workflow that produces runnable code. You must:

1. **Implement the test plan** from `TESTS.md`.
2. **Run the tests against the legacy** until they pass — this proves they capture current behavior, not aspirational behavior.
3. **Port the module** to the target stack, copying business logic as faithfully as possible (parity-first, not idiomatic-first).
4. **Run the tests against the new implementation** until they pass.
5. **Stop there.** Refactoring for idiomatic style is a separate, post-parity step that the user kicks off explicitly.

## Procedure

### 1. Load configuration and all prior artifacts

Read:

- `.rdd.yml` — every field
- `{artifacts_dir}/MAP.md`
- `{artifacts_dir}/{module}/SPEC.md`
- `{artifacts_dir}/{module}/TESTS.md` — **required**

If any artifact is missing, instruct the user to run the corresponding upstream skill first. Do not proceed.

Confirm with the user that the test plan is reviewed and approved before writing any code.

### 2. Set up the test harness

Per the harness plan in `TESTS.md`:

- Install or configure the test framework (`target.test_framework`).
- Wire up the test database (testcontainers or equivalent).
- Build the dual-target client abstraction (legacy and new behind the same interface).
- Configure auth (mint test JWTs that both systems accept — usually means the new system validates the same JWT as legacy during migration).
- Configure deterministic time and random seeds where the spec requires it.

Commit this setup as a discrete, reviewable change.

### 3. Implement the tests against the LEGACY system first

Order: spec'd → property-based → boundary → side-effect.

For each test in `TESTS.md`:

- Implement it pointing at the legacy implementation.
- Run it. **Expect it to pass.** If it fails, you have one of three situations:
  - **Test is wrong** — fix the test
  - **Spec is wrong** — update `SPEC.md`, then fix the test
  - **Legacy has a bug the spec didn't capture** — discuss with user; default is to update spec to match legacy (parity), mark the BR as a known bug
- Never adjust the legacy code to make tests pass. The legacy is the source of truth at this stage.

When all tests are green against legacy, commit. This is the **behavior lock** — from here, parity is mechanically verifiable.

### 4. Scaffold the new module in the target stack

Create the module under `target.source` following the conventions of the target stack:

- Module/package directory with the chosen name
- Entry points matching the spec's API surface (one route per spec endpoint)
- Empty implementations that return 501 or throw "not implemented"
- Any required dependency injection / module registration

At this point, the new endpoints exist but do nothing. Tests against the new target should fail cleanly.

### 5. Port the business logic — parity-first

For each entry point, port the legacy logic to the new module:

- **Copy the structure**, adapting only for syntax differences (e.g., Deno → Node, JS → TS, framework idioms required to compile).
- **Keep the same control flow.** Same `if` order, same early returns, same error responses. Even if it looks ugly.
- **Keep the same database queries** when possible. If the target stack uses a different ORM, translate queries 1:1, not "the way the new ORM prefers".
- **Keep the same field names** in responses. Don't camelCase what was snake_case, etc.
- **Keep bugs that callers may depend on.** Spec said `(legacy bug, intentional parity)` — honor it.

Run the dual-target tests after each entry point. Iterate until green.

### 6. Handle integration boundaries carefully

For external services (Stripe, payment gateways, message brokers, AI providers):

- Use **the same SDK or HTTP client choice** initially, even if the new stack has a "better" wrapper.
- Pin **the same API version**.
- Mirror **the same retry policy and timeouts**.
- Idempotency keys: if the legacy generates them a certain way, generate them the same way.

Improvement happens after parity, not during.

### 7. Verify side effects

Side-effect tests in `TESTS.md` must pass on both targets. Pay extra attention:

- **Webhook payloads must match byte-for-byte** (modulo timestamps and ids). Consumers of the webhook should not see a difference after cutover.
- **Email/notification content** must match.
- **Audit log entries** must match shape.
- **Queue messages** must match shape and routing key.

If any side effect cannot match exactly (e.g., the new logger formats differently), document the deviation in `SPEC.md` under a new section "Intentional deviations from legacy".

### 8. Run the full suite against both targets

```
TEST_TARGET=legacy <test-command>
TEST_TARGET=new <test-command>
```

Both must be green. If a test passes on legacy but fails on new, you have a parity gap — fix the new code, not the test.

If a test passes on new but fails on legacy, you found a legacy bug that you accidentally fixed. Decide with the user: revert (parity) or document (intentional deviation).

### 9. Stop. Do not refactor yet.

This is the hardest step. The new code is parity-correct but probably not idiomatic. Stop anyway.

Refactoring for style is a **separate session** the user initiates explicitly. The tests from this skill will guard that refactor — that's their second job.

### 10. Hand-off

Tell the user:

- Tests passing on legacy: count
- Tests passing on new: count
- Any deviations recorded
- Next steps in the user's hands:
  - Cutover (feature flag, gradual rollout, monitoring)
  - Idiomatic refactor (separate session, behavior-locked by these tests)
  - Cleanup (delete legacy code after stable cutover)

## Anti-patterns to avoid

- **Don't refactor while porting.** "Improving while you're already there" turns a parity port into a behavior change. Two PRs, not one.
- **Don't translate to "idiomatic" early.** Even if the legacy code is bad, copy it shape-by-shape. You'll clean up after the tests prove the new is equivalent.
- **Don't skip the legacy-side test run.** Tests that were never validated against legacy might pass against new for the wrong reason.
- **Don't merge the cutover into the port.** This skill produces a parity-correct new module behind a feature flag. Flipping the flag is a separate, observability-gated decision.
- **Don't trust "looks the same" — run the suite.** Tiny differences (rounding, default values, sort order) bite in production.

## When to spawn sub-agents

For long porting tasks (>1000 lines of code to translate), spawn the appropriate code-writing agent for the target stack with explicit instructions: "parity-first, copy structure, do not refactor". Pass it the legacy file and the spec section. Verify its output against the test suite before accepting.
