---
name: rdd-port
description: Use this skill after rdd-tests to implement the characterization test suite, run it against the legacy system, then port the module to the target stack with parity. Triggers on "port X", "rewrite X with parity", "implement the migration of X". Produces working code + green tests on both legacy and new.
---

# rdd-port — Implement tests, then port with parity

You are executing the rewrite. This is the only RDD skill that produces runnable code. You must:

1. **Implement the test plan** from `TESTS.md`
2. **Run the tests against the legacy** until they pass — this proves they capture current behavior, not aspirational behavior
3. **Port the module** to the target stack, copying business logic faithfully (parity-first, not idiomatic-first)
4. **Run the tests against the new implementation** until they pass
5. **Stop there.** Refactoring for idiomatic style is a separate session the user kicks off explicitly.

The single most common failure mode of this skill is rushing through entry points without verifying parity at each step. Treat the per-entry-point STOP as a hard terminator.

---

## Modes

- **Default (pause)**: stop after each entry point port, emit a completion line, ask `Continuar para <next>?`, and **wait** for the user's reply in a new turn.
- **Continuous (autopilot)**: only active when the user **explicitly** requested it at session start with phrases like "execute tudo", "autopilot", "don't pause between entry points", "run all at once". When unsure, default to pause.

---

## Inputs

The user invokes the skill with a module name (e.g., "/rdd-port products"). Resolve:

- `{artifacts_dir}/TARGET.md` — must exist
- `{artifacts_dir}/MAP.md` — must exist
- `{artifacts_dir}/{module}/SPEC.md` — must exist
- `{artifacts_dir}/{module}/TESTS.md` — must exist

If any is missing, instruct the user to run the upstream skill first. **Do not proceed.**

---

## Progress file

A progress file persists state across sessions. It lives at `{artifacts_dir}/{module}/PORT.progress.md`:

```markdown
# {module} — Port Progress

**Status:** in_progress | completed
**Phase:** preflight | harness | lock_legacy | scaffold | port_entrypoints | side_effects | final_verification | done
**Entry points:** X/Y completed

## Phases

### preflight
- **Status:** completed | pending
- **Notes:** ...

### harness
- **Status:** completed | pending
- **Notes:** ...

### lock_legacy
- **Status:** completed | pending
- **Tests passing on legacy:** X/Y
- **Notes:** ...

### scaffold
- **Status:** completed | pending
- **Notes:** ...

### port_entrypoints

#### EP-1: <name>
- **Status:** completed | pending
- **Tests on new:** X/Y passing
- **Observations:** out-of-scope notes or "none"

(repeat per entry point)

### side_effects
- **Status:** completed | pending
- **Notes:** ...

### final_verification
- **Status:** completed | pending
- **Full suite:** pass | fail
- **Type check:** pass | fail
- **Build:** pass | fail
```

This file is the source of truth for resuming and for the final completion report.

---

## Procedure

### Phase 0 — Preflight

Before touching code, check:

1. **Branch check.** Run `git status` and `git branch --show-current`. If on `main` or default branch, or if there are uncommitted changes touching files outside the migration scope, **stop** and ask the user to set up the right branch.
2. **Target subproject readiness.** `target.source` directory exists. Dependencies installed (`node_modules` or equivalent present). If not, ask the user to set it up first.
3. **Plan sanity.** `SPEC.md` and `TESTS.md` parse correctly: BRs numbered, tests in TESTS.md cover all BRs (or unmapped BRs are explicitly acknowledged). Coverage matrix is complete.
4. **Resume check.** Look for `PORT.progress.md`. If found, read it: which phase to resume from, which entry points are completed. Tell the user: *"Found progress file: phase=`port_entrypoints`, EP X/Y completed. Resuming from EP-N."*
5. **Mode check.** Did the user request continuous mode at session start? Default = pause.

If preflight passes, create or update `PORT.progress.md` (fresh start: all phases pending; resume: untouched, just read).

Then create the persistent task list (one `TaskCreate` per phase/entry point, in execution order):

- Setup harness
- Lock legacy behavior
- Scaffold new module
- Port EP-1: <name>
- Port EP-2: <name>
- ...
- Verify side effects
- Final verification

All tasks start `pending`. As you start each phase/EP, flip to `in_progress` (immediately before the work). Mark `completed` at end of the phase, just before the STOP point.

### Phase 1 — Set up the test harness

Per the harness plan in `TESTS.md`:

- Install/configure the test framework (`target.test_framework`)
- Wire up the test database (testcontainers or equivalent)
- Build the dual-target client abstraction (legacy and new behind same interface)
- Configure auth (mint test JWTs both systems accept)
- Configure deterministic time and random seeds where the spec requires

Commit this as a discrete, reviewable change.

Update `PORT.progress.md`: `harness` → `completed`. Then **STOP** (default mode) and ask `Continuar para lock_legacy?`.

### Phase 2 — Lock legacy behavior

Implement every test in `TESTS.md` pointing at the **legacy** implementation. Run them. **Expect them to pass.**

If a test fails, you have one of three situations:

- **Test is wrong** → fix the test
- **Spec is wrong** → update `SPEC.md`, then fix the test
- **Legacy has a bug the spec didn't capture** → discuss with user; default is to update `SPEC.md` to match legacy (parity), mark the BR as `(legacy bug, intentional parity)`

Never adjust the legacy code to make tests pass. The legacy is the source of truth at this stage.

When all tests are green against legacy, commit. **This is the behavior lock** — from here, parity is mechanically verifiable.

Update `PORT.progress.md`: `lock_legacy` → `completed`, record `Tests passing on legacy: X/Y`. **STOP** and ask `Continuar para scaffold?`.

### Phase 3 — Scaffold the new module

Create the module under `target.source` following `TARGET.md` conventions:

- Module/package directory
- Entry points matching the spec's API surface (one route per spec endpoint)
- Empty implementations that return 501 or throw "not implemented"
- Required dependency injection / module registration

The new endpoints exist but do nothing. Tests against the new target should fail cleanly.

Commit. Update `PORT.progress.md`: `scaffold` → `completed`. **STOP** and ask `Continuar para port_entrypoints?`.

### Phase 4 — Port entry points (the per-EP loop)

For each entry point in order (read from `SPEC.md` API surface), execute these steps. **Do not batch.**

#### Step 4.1 — Plan the EP

Re-read the EP's section in `SPEC.md` (request, response, BRs that govern it, side effects, error codes) and the corresponding tests in `TESTS.md`. Keep an internal checklist for this EP: one item per BR + one item per side effect + "run tests on new".

Mark this EP's task `in_progress`.

#### Step 4.2 — Port the logic (parity-first)

- **Copy the structure** from legacy, adapting only for syntax differences (Deno → Node, JS → TS, framework idioms required to compile)
- **Keep the same control flow.** Same `if` order, same early returns, same error responses. Even if it looks ugly.
- **Keep the same database queries.** If the target ORM differs, translate queries 1:1 — not "the way the new ORM prefers"
- **Keep the same field names** in responses. Don't camelCase what was snake_case
- **Keep bugs that callers depend on.** Spec said `(legacy bug, intentional parity)` — honor it

Stay within scope: only touch files required by **this EP**. If you notice unrelated issues (dead code, formatting, refactoring opportunities), record them in `PORT.progress.md` under this EP's `Observations` and do **not** act on them.

#### Step 4.3 — Run only this EP's tests against the new target

```
TEST_TARGET=new <test-command for this EP's tests>
```

Not the full suite. Save that for final verification.

#### Step 4.4 — Fix loop (max 3 attempts)

If tests pass on first run, proceed to step 4.5.

If tests fail, enter the fix loop. Read the failure output, diagnose root cause, apply a focused fix, re-run **the same tests**. Maximum **3 attempts**. Count deliberately.

**Discipline:**

- **Read the error.** Don't retry blindly. Same fix applied twice = wrong diagnosis.
- **Fix the root cause.** Don't weaken tests to make them pass. Don't add `.skip`, `.only`, `xit`. Don't catch and swallow errors to hide them.
- **Stay in scope.** If the failure reveals a problem in a previously-completed EP's code, **stop** — that's escalation, not quietly editing a finished EP.
- **No shortcuts.** Never disable hooks. Never bypass safety checks.

After 3 unsuccessful attempts, **stop**. Report:

- Which EP is stuck
- Concise test failure output
- Your hypothesis about root cause
- What you've tried (each of the 3 attempts)

Wait for user guidance. Do not proceed.

#### Step 4.5 — Run the same tests against legacy (parity check)

```
TEST_TARGET=legacy <test-command for this EP's tests>
```

Both must be green. If a test passes on `new` but fails on `legacy`, you found a legacy bug your port accidentally fixed. Decide with the user:

- **Revert** to match legacy (default — parity)
- **Document** as `Intentional deviations from legacy` in `SPEC.md` and keep the fix

Don't decide unilaterally — flag it.

#### Step 4.6 — STOP after the EP

Update the EP's task to `completed`. Update `PORT.progress.md`: this EP's status → `completed`, record test counts, fold any out-of-scope observations.

These are the **last tool calls allowed before the stop**. Then emit:

```
EP-N (<name>) ported. Tests on new: X/Y passing. Tests on legacy: X/Y passing.
Continuar para EP-N+1?
```

**STOP.** Do not call any further tools. Do not start reading the next EP's files. Do not flip the next EP's task to `in_progress`. Wait for the user's reply in a new turn.

In **continuous mode**, skip the question and the STOP — go straight to step 4.1 of the next EP. Otherwise, default to pause.

When the last EP is done, skip the question and the STOP entirely (the per-EP report is folded into the phase-level Completion report). Proceed to Phase 5.

### Phase 5 — Verify side effects

Run all side-effect tests from `TESTS.md` against the **new** target. Pay extra attention to:

- **Webhook payloads** — must match byte-for-byte (modulo timestamps, ids). External consumers should not see a difference after cutover.
- **Email/notification content** — must match.
- **Audit log entries** — must match shape.
- **Queue messages** — must match shape and routing key.

If any side effect cannot match exactly (e.g., new logger formats differently), document under `Intentional deviations from legacy` in `SPEC.md`.

Update `PORT.progress.md`: `side_effects` → `completed`. Proceed to Phase 6 (no STOP — final verification is the natural close).

### Phase 6 — Final verification

Run the phase-level checks:

1. **Full test suite** against `TEST_TARGET=new` — every test for this module, not just per-EP
2. **Full test suite** against `TEST_TARGET=legacy` — confirm legacy still green (didn't regress while you were editing the dual-target harness)
3. **Type-check** the target subproject (e.g., `tsc --noEmit`)
4. **Build** the target subproject (e.g., the production build command)

If any check fails, apply the same fix-loop discipline (max 3 attempts shared across all failing checks). Stop and report if still failing after 3.

When all green, update `PORT.progress.md`: `final_verification` → `completed`, `Status` → `completed`, `Phase` → `done`.

### Phase 7 — Stop. Do not refactor yet.

This is the hardest discipline step. The new code is parity-correct but probably not idiomatic. **Stop anyway.**

Refactoring for style is a **separate session** the user initiates explicitly. The tests from this skill will guard that refactor — that's their second job.

---

## Completion report

When the module is fully ported, generate the report:

- Tests passing on legacy: X/Y
- Tests passing on new: X/Y
- Type-check, build status
- Any intentional deviations recorded in `SPEC.md` (list)
- Out-of-scope observations aggregated from `PORT.progress.md` (as a list of follow-ups for the user, not actions to take)

Next steps in the user's hands:

- **Cutover** (feature flag, gradual rollout, monitoring)
- **Idiomatic refactor** (separate session, behavior-locked by these tests)
- **Cleanup** (delete legacy code after stable cutover)

Git operations (commit, push, PR) are out of scope — the user owns version control. You commit at the points specified above (end of each phase) but do not push or open PRs.

---

## Anti-patterns to avoid

- **Don't refactor while porting.** "Improving while you're already there" turns a parity port into a behavior change. Two sessions, not one.
- **Don't translate to "idiomatic" early.** Even if legacy code is bad, copy it shape-by-shape. Clean up after the tests prove the new is equivalent.
- **Don't skip the legacy-side test run** (Phase 2 and step 4.5). Tests never validated against legacy might pass on new for the wrong reason.
- **Don't merge cutover into the port.** This skill produces a parity-correct module behind a feature flag. Flipping the flag is a separate, observability-gated decision.
- **Don't trust "looks the same" — run the suite.** Tiny differences (rounding, defaults, sort order) bite in production.
- **Don't violate the STOP.** "Continuar para EP-N+1?" is a terminator, not a rhetorical question.
- **Don't lose count in the fix loop.** 3 attempts means 3, not "until I figure it out".
- **Don't quietly edit a completed EP.** If a failure points back to one, escalate.
- **Don't push to remote.** User owns git operations beyond local commits.

---

## When to spawn sub-agents

For long porting tasks (>1000 lines per EP), spawn the appropriate code-writing agent for the target stack with explicit instructions: "parity-first, copy structure, do not refactor". Pass it the legacy file and the spec section. **Verify its output against the test suite** before accepting. Do not delegate the fix loop — run that yourself with full context.
