---
name: rdd-improve-05
description: Use this skill after rdd-refactor-04 to refactor a parity-correct module for idiomatic style and quality, with the existing characterization tests as a parity safety net. Fifth (optional) step in the RDD pipeline (per-module). Triggers on "improve X", "refactor X for style", "clean up X after refactoring", "make X idiomatic". Produces idiomatic code + IMPROVE.progress.md, tests stay green throughout.
---

# rdd-improve-05 — Idiomatic refactor with parity safety net

You are improving the **internal quality** of a module that was ported with parity (`/rdd-refactor-04`) but isn't idiomatic. The characterization tests from `/rdd-refactor-04` are your safety net: any refactor that changes observable behavior fails a test, and you revert or fix.

This skill runs **after** the module has been ported and parity is verified. It does not run on legacy code, and it does not run on code that hasn't been characterized — without the tests there is no safety net, and "refactoring" becomes "rewriting from memory".

---

## Modes

- **Default (pause)**: stop after each refactor unit, emit a completion line, ask `Continuar para R-N+1?`, and **wait** for the user's reply in a new turn.
- **Continuous (autopilot)**: only when the user **explicitly** requested it at session start. When unsure, default to pause.

---

## Inputs

The user invokes the skill with a module name (e.g., "/rdd-improve-05 products"). Resolve:

- `{artifacts_dir}/TARGET.md` — for chosen conventions
- `{artifacts_dir}/{module}/SPEC.md` — for the contract that must remain honored
- `{artifacts_dir}/{module}/TESTS.md` — for the test plan
- `{artifacts_dir}/{module}/REFACTOR.progress.md` — must show `Status: completed`. If not, instruct the user to finish the port first.
- The new code under `target.source` for this module

---

## Progress file

`{artifacts_dir}/{module}/IMPROVE.progress.md`:

```markdown
# {module} — Improve Progress

**Status:** in_progress | completed
**Phase:** preflight | identify | refactor | final_verification | done
**Refactors:** X/Y completed

## Phases

### preflight
- **Status:** completed | pending
- **Tests green at start:** Y/Y on new

### identify
- **Status:** completed | pending
- **IMPROVE.md generated:** rdd/{module}/IMPROVE.md
- **Refactors planned:** N

### refactor

#### R-1: <name>
- **Status:** completed | reverted | pending
- **Tests after:** Y/Y on new
- **Notes:** what changed, any observations

(repeat per refactor)

### final_verification
- **Status:** completed | pending
- **Full suite:** pass | fail
- **Type check:** pass | fail
- **Build:** pass | fail
```

---

## Procedure

### Phase 0 — Preflight

Before suggesting a single refactor:

1. **Port phase complete.** `REFACTOR.progress.md` shows `Status: completed` and `final_verification` passed. If not, **stop** and instruct the user to finish `/rdd-refactor-04 {module}` first.
2. **Tests green now.** Run all tests for this module against `TEST_TARGET=new`. They must all pass before you start. If any fail, **stop** — the safety net has a hole; user must fix before improving.
3. **Branch check.** `git status` clean (or only changes inside the module's scope); current branch is not `main` / default.
4. **Resume check.** Look for `IMPROVE.progress.md`. If found, resume from the right refactor. Tell the user: *"Found progress file: refactor R-N of M completed. Resuming from R-N+1."*
5. **Mode check.** Did the user request continuous mode? Default = pause.

If preflight passes, create or update `IMPROVE.progress.md`. Create the persistent task list with `TaskCreate` (one task per refactor unit, plus identify and final_verification).

### Phase 1 — Identify improvements

Read the new code for this module under `target.source`. Compare against `TARGET.md` conventions. Look for:

- **Conventions violations** — naming, folder structure, error handling style, validation pattern, log format
- **Direct legacy translations that don't fit** the new stack's idioms (e.g., callback-style code in an async/await stack, manual error checks where typed exceptions would be cleaner)
- **Duplication** that an extracted function or value object would eliminate
- **Missing abstractions** — money handled as numbers instead of `Money` value object, raw timestamps instead of `Date`, etc.
- **Dead code** copied from legacy that the new code path doesn't reach
- **Implicit coupling** — modules talking through shared globals instead of explicit dependencies
- **Test code that violates `/rdd-refactor-04` boundary rules** — tests asserting implementation rather than behavior. Flag for deletion or rewrite.

For each candidate refactor, classify by **risk**:

- **Low risk (safe refactors)**: rename, extract function, inline variable, replace type with type alias, reorganize imports
- **Medium risk**: extract value object, replace conditional with polymorphism, consolidate duplicated logic
- **High risk**: change error handling pattern, restructure module boundaries, replace persistence pattern

Order: low risk first, high risk last. High-risk refactors should be small enough to revert if they break a test.

**Write `{artifacts_dir}/{module}/IMPROVE.md`** using `templates/IMPROVE.md`. List each refactor with ID (R-NN), description, risk, expected benefit, scope (which files), and parity assertion (which tests must remain green).

The user reviews `IMPROVE.md` before you apply any refactor. **Wait for explicit approval** before moving to phase 2.

### Phase 2 — Refactor loop (per refactor unit)

For each refactor R-NN in order, execute these steps. **Do not batch.**

#### Step 2.1 — Plan the refactor

Mark the refactor's task `in_progress`. Re-read R-NN in `IMPROVE.md`. Confirm the scope (files touched, expected delta). If scope grew during identify, push the larger version to a follow-up R-NN+M and keep this one small.

#### Step 2.2 — Apply the refactor

Make the change. Touch only files in R-NN's scope. If a refactor "wants" to grow (changing one thing reveals another that needs changing), **stop and split** — finish the current small step, then propose a new R-NN+M.

#### Step 2.3 — Run the module's tests

```
TEST_TARGET=new <test-command for this module>
```

Not the full project suite. Save that for final verification.

#### Step 2.4 — Decide based on result

**Green:** the refactor preserved observable behavior. Commit. Update `IMPROVE.progress.md`: R-NN → `completed`, record test count. Proceed to step 2.5.

**Red:** the refactor changed observable behavior. Three options, in order of preference:

1. **Revert.** Undo the refactor entirely. Mark R-NN → `reverted` in progress file with reason. Move to next refactor. *(Default — preserves safety net intact.)*
2. **Fix the refactor** (if you can do so within 3 attempts). Apply the fix-loop discipline from `/rdd-refactor-04` step 4.4: read error, diagnose, focused fix, re-run. Max 3 attempts. After 3 failures, revert.
3. **Update the spec/tests** (rare; only if the refactor reveals that the legacy behavior the test locks is itself a bug AND the user explicitly approves changing it). This is no longer pure refactoring — it's an intentional deviation. Mark in `SPEC.md` under "Intentional deviations from legacy".

**Never weaken tests to make them pass.** Never `.skip`, `.only`, `xit`. Never catch and swallow errors. The whole point of `/rdd-improve-05` is the safety net — undermining it defeats the skill.

#### Step 2.5 — STOP after the refactor

These are the **last tool calls** in default mode: `TaskUpdate` to mark this refactor `completed` (or `reverted`), update `IMPROVE.progress.md`. Then emit:

```
R-NN (<name>) <completed|reverted>. Tests: Y/Y on new.
Continuar para R-NN+1?
```

**STOP.** Wait for the user's reply.

In continuous mode, skip the question and proceed to step 2.1 of the next refactor.

When the last refactor is done, skip the question and proceed to Phase 3.

### Phase 3 — Final verification

Run:

1. **Full module test suite** against `TEST_TARGET=new` — every test for this module
2. **Full module test suite** against `TEST_TARGET=legacy` — confirm legacy still green (didn't regress while editing the harness)
3. **Type-check** the target subproject
4. **Build** the target subproject

Apply the 3-attempt fix-loop if anything fails. Stop and report if still failing after 3.

When all green, update `IMPROVE.progress.md`: `final_verification` → `completed`, `Status` → `completed`, `Phase` → `done`.

---

## Completion report

When the module is fully improved:

- Refactors applied: X
- Refactors reverted (and why): Y
- Tests remained green throughout: confirmed
- Type-check, build status
- Any intentional deviations from legacy added during this skill (list)

Next steps in the user's hands:

- Re-run `/rdd-improve-05 {module}` later if more refactors are identified
- Cutover (if not already done) — same flag flip as `/rdd-refactor-04` Phase 5

---

## Anti-patterns to avoid

- **Don't refactor without the safety net.** If `/rdd-refactor-04` isn't complete, do not run this skill.
- **Don't grow refactors mid-application.** Small steps that revert cleanly are the whole point.
- **Don't disable tests to "make them pass after refactoring".** A failing test means the refactor changed behavior — revert or document deviation.
- **Don't refactor the test code while refactoring the production code.** Two separate sessions if you must.
- **Don't push high-risk refactors first.** Order by risk; build confidence.
- **Don't skip the STOP between refactors.** Each is a separable commit with a clear rollback boundary.
- **Don't run the full project suite after each refactor.** Module tests are the per-refactor signal; full suite is final verification only.

---

## When to spawn sub-agents

For long refactors (>500 lines transformed in one R-NN), spawn the appropriate code agent for the target stack with explicit parity instructions: "preserve all observable behavior; the test suite must stay green". Pass it the original code, the refactor goal, and the test files. **Verify against the suite** before accepting its output. Do not delegate the fix loop.
