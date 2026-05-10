---
name: rdd-improve-05
description: Use this skill after rdd-refactor-04 to refactor a parity-correct module for idiomatic style and quality, with the existing characterization tests as a parity safety net. Fifth (optional) step in the RDD pipeline (per-module). Triggers on "improve X", "refactor X for style", "clean up X after refactoring", "make X idiomatic". Produces idiomatic code + improve/IMPROVE.md + improve/NN_&lt;name&gt;.md per refactor, updates the module's PROGRESS.md; tests stay green throughout.
---

# rdd-improve-05 — Idiomatic refactor with parity safety net

You are improving the **internal quality** of a module that was ported with parity (`/rdd-refactor-04`) but isn't idiomatic. The characterization tests from `/rdd-refactor-04` are your safety net: any refactor that changes observable behavior fails a test, and you revert or fix.

This skill runs **after** the module has been ported and parity is verified. It does not run on legacy code, and it does not run on code that hasn't been characterized — without the tests there is no safety net, and "refactoring" becomes "rewriting from memory".

This skill runs **autonomously**. After pre-flight passes (and the upstream-port gate is satisfied), the skill identifies refactors, applies them in risk order (low → high), and proceeds — no per-refactor confirmation prompts. The safety net is the test suite: a refactor that breaks tests is reverted automatically (parity-first default). The skill stops only when it cannot continue:

- **Pre-flight fails** (port not complete, tests not green at start, dirty branch, upstream gate blocked) — hard fail, exit
- **Fix-loop exhausts 3 attempts** on the same refactor — automatic revert, log the attempt, continue with the next refactor
- **Final verification fails after 3 fix attempts** — hard fail, exit

---

## Inputs

The user invokes the skill with a module name (e.g., "/rdd-improve-05 products"). Resolve:

- `{artifacts_dir}/TARGET.md` — for chosen conventions
- `{artifacts_dir}/MAP.md` — to look up the module's **`Seq`** and find its `module_dir`
- `{module_dir}/spec/SPEC.md` — for the contract that must remain honored. `module_dir` is `{artifacts_dir}/<Seq>_<module>/` (e.g., `rdd/003_products/`); accept either bare name or prefixed form as input.
- `{module_dir}/spec/TESTS.md` — for the test plan
- `{module_dir}/PROGRESS.md` — must show pipeline `port` row as `✅ completed` and `final` row as `✅ completed`. If not, instruct the user to finish the port first.
- The new code under `target.source` for this module

### Module directory resolution

Same three-form input as `/rdd-specify-03` and `/rdd-refactor-04`: bare name (`products`), prefixed (`003_products`), or Seq only (`003`). Glob `{artifacts_dir}/[0-9][0-9][0-9]_<module>/` to resolve, expect exactly one match (the directory was created by `/rdd-specify-03` and populated by `/rdd-refactor-04`). Backward compat: legacy flat layout accepted with a one-line warning.

---

## Module layout (this skill writes here)

```
{module_dir}/
├── PROGRESS.md          ← updated continuously: improve pipeline row, refactors table, history
├── spec/                ← read-only here
├── tasks/               ← read-only here (port artifacts)
└── improve/             ← created lazily on first run of this skill
    ├── IMPROVE.md       ← Phase 1 output (refactor plan)
    ├── 01_<name>.md     ← Phase 2 output (one file per refactor unit)
    ├── 02_<name>.md
    └── ...
```

Per-refactor files use `templates/task.md` shape (status, source/target, tests, observations) — the same template `/rdd-refactor-04` uses for port tasks. The numeric prefix is the order of application (low-risk first).

---

## Progress

This skill updates `{module_dir}/PROGRESS.md` (the same aggregate file `/rdd-refactor-04` writes to). The Pipeline table's `improve` row goes from `⬜` → `🔄 in_progress` → `✅ completed`. The Improve refactors table is populated as Phase 1 plans them and updated as each is applied/reverted. Per-refactor detail (test counts, observations, fix-loop attempts) lives in `improve/NN_<name>.md`.

There is no separate `IMPROVE.progress.md` — `PROGRESS.md` is the single source of truth.

---

## Procedure

### Phase 0 — Preflight

Before suggesting a single refactor:

1. **This module's port complete.** `{module_dir}/PROGRESS.md` shows pipeline `port` row as `✅ completed` AND `final` row as `✅ completed`. If not, **stop** and instruct the user to finish `/rdd-refactor-04 {module}` first.
2. **Upstream-port gate (HARD BLOCK).** Same gate as `/rdd-refactor-04` Phase 0. Improve cannot run while any upstream module's port is incomplete and not explicitly skipped — refactoring against an incomplete dependency tree wastes work, since interfaces may shift when upstreams land.

   Read `MAP.md`, get `current_seq`. For every directory `{artifacts_dir}/[0-9][0-9][0-9]_*/` with prefix `< current_seq`:
   - **OK** if it has `PROGRESS.md` with pipeline `port` row marked `✅ completed`
   - **OK** if `.rdd.yml` lists the module under `skipped:` (schema documented in `/rdd-refactor-04` Phase 0)
   - **NOT OK** otherwise — emit the same structured error as `/rdd-refactor-04`, listing pending upstreams, and **stop**.

3. **Tests green now.** Run all tests for this module against `TEST_TARGET=new`. They must all pass before you start. If any fail, **stop** — the safety net has a hole; user must fix before improving.
4. **Branch check.** `git status` clean (or only changes inside the module's scope); current branch is not `main` / default.
5. **Resume check.** Read the Improve refactors table in `PROGRESS.md`. If any rows are `🔄 in_progress` or `⬜ pending`, resume from the lowest-numbered pending. Print one line: *"Resuming from refactor `NN_<name>`, M of Total complete."* Then proceed.

If preflight passes, set the pipeline `improve` row in `PROGRESS.md` to `🔄 in_progress`. Create the `improve/` subdirectory if it doesn't exist. Create the persistent task list with `TaskCreate` (one task per refactor unit named like the file — `Improve: <refactor-name>` — plus `Identify refactors` and `Final verification`).

### Phase 1 — Identify improvements

**Primary input: `spec/SPEC.md` "Improvement candidates" section.** Each `IC-NN` entry is a bug, quirk, code-smell, or design suggestion that `/rdd-specify-03` and `/rdd-refactor-04` flagged during the parity-locked port. They are the high-priority candidates for this phase — the user already saw them and they survived the parity bar. Translate each `IC-NN` into one or more refactors here.

**Secondary input: read the new code** under `target.source`. Compare against `TARGET.md` conventions and look for:

- **Conventions violations** — naming, folder structure, error handling style, validation pattern, log format
- **Direct legacy translations that don't fit** the new stack's idioms (e.g., callback-style code in an async/await stack, manual error checks where typed exceptions would be cleaner)
- **Duplication** that an extracted function or value object would eliminate
- **Missing abstractions** — money handled as numbers instead of `Money` value object, raw timestamps instead of `Date`, etc.
- **Dead code** copied from legacy that the new code path doesn't reach
- **Implicit coupling** — modules talking through shared globals instead of explicit dependencies
- **Test code that violates `/rdd-refactor-04` boundary rules** — tests asserting implementation rather than behavior. Flag for deletion or rewrite.

**Behavior-changing improvements (bugs from `IC-NN` with class `bug`):** these change observable behavior and therefore can break tests. Treat them as **medium or high risk** by default. The fix-or-revert flow (Step 2.4) handles them: if a fix breaks the safety net, the refactor is reverted and the user can decide later (out of scope for autonomous mode). Behavior-preserving improvements (`code-smell`, `design`) are **low risk**.

For each candidate refactor, classify by **risk**:

- **Low risk (safe refactors)**: rename, extract function, inline variable, replace type with type alias, reorganize imports
- **Medium risk**: extract value object, replace conditional with polymorphism, consolidate duplicated logic
- **High risk**: change error handling pattern, restructure module boundaries, replace persistence pattern

Order: low risk first, high risk last. High-risk refactors should be small enough to revert if they break a test.

**Write `{module_dir}/improve/IMPROVE.md`** using `templates/IMPROVE.md`. List each refactor with name (used as the file prefix `NN_<name>.md`), description, **source `IC-NN` (if any)**, risk, expected benefit, scope (which files), and parity assertion (which tests must remain green). Order by risk, low → high; the order determines the `NN` prefix. Refactors derived from a SPEC `IC-NN` cite that ID; refactors derived from convention review cite "convention review".

Initialize the Improve refactors table in `PROGRESS.md` with one row per planned refactor (status `⬜ pending`, file path `improve/NN_<name>.md` not yet created).

Print one line summarizing the plan (e.g., `"Refactors planned: 7 (5 low-risk, 2 medium-risk, 0 high-risk). Path: {module_dir}/improve/IMPROVE.md"`) and proceed directly to Phase 2. The user can intervene afterward by editing `IMPROVE.md` and re-running, or by reverting commits — but the skill does not stop to ask for approval.

### Phase 2 — Refactor loop (per refactor unit)

For each refactor R-NN in order, execute these steps. **Do not batch.**

#### Step 2.1 — Open the refactor file

Mark the refactor's `TaskCreate` entry `in_progress`. Create `{module_dir}/improve/NN_<name>.md` from `templates/task.md` (or re-open if it exists from a prior session). Re-read this refactor's section in `improve/IMPROVE.md`. Confirm the scope (files touched, expected delta). If scope grew during identify, push the larger version to a follow-up `NN+M_<name>.md` and keep this one small. Update the matching row in `PROGRESS.md`'s Improve table to `🔄 in_progress`.

#### Step 2.2 — Apply the refactor

Make the change. Touch only files in this refactor's scope. If a refactor "wants" to grow (changing one thing reveals another that needs changing), **stop and split** — finish the current small step, then add a new `improve/NN+M_<name>.md` entry to the queue.

#### Step 2.3 — Run the module's tests

```
TEST_TARGET=new <test-command for this module>
```

Not the full project suite. Save that for final verification.

#### Step 2.4 — Decide based on result (autonomous)

**Green:** the refactor preserved observable behavior. Commit. Finalize the refactor file (`Status: completed`, test counts, observations). Mirror the row in `PROGRESS.md`'s Improve table to `✅ completed`. Proceed to step 2.5.

**Red:** the refactor changed observable behavior. Apply the fix-loop discipline from `/rdd-refactor-04`: read error, diagnose, focused fix, re-run. Each attempt appends a one-line entry under `Fix-loop attempts` in this refactor file. **Max 3 attempts**.

- **Within 3 attempts the refactor turns green** → commit, finalize as `completed`, mirror to `PROGRESS.md`, proceed.
- **After 3 attempts still red** → **revert** the refactor (`git restore` / `git reset` to before this refactor began), set the refactor file's `Status: reverted` with the failing test name and the diagnostic notes from the attempts. Mirror to `PROGRESS.md`'s Improve table as `⏭ reverted`. **Continue to the next refactor** — a single failed refactor doesn't stop the loop; it just doesn't get applied.

The user reviews the `reverted` list at the end. They can re-run the skill later, hand-craft the failing refactor in a new session, or accept that it's genuinely incompatible with the safety net.

The "update spec/tests to allow the refactor" path is **out of scope for autonomous mode** — it's a behavior change disguised as a refactor and requires deliberate user input. If a refactor reveals a legacy bug worth fixing, the user does that in a separate session by editing `spec/SPEC.md` first, then re-running `/rdd-refactor-04`.

**Never weaken tests to make them pass.** Never `.skip`, `.only`, `xit`. Never catch and swallow errors. The whole point of `/rdd-improve-05` is the safety net — undermining it defeats the skill.

#### Step 2.5 — Log progress and continue

Print one progress line using the **refactor's name** and proceed directly to step 2.1 of the next refactor:

```
Refactor <name> (X/Y) <completed|reverted>. Tests: Y/Y on new.
```

Example: `Refactor extract-money-vo (1/7) completed. Tests: 45/45 on new.`

When the last refactor is done, set the pipeline `improve` row's detail to `Y/Y refactors` (showing how many were applied vs reverted) and proceed to Phase 3.

### Phase 3 — Final verification

Run:

1. **Full module test suite** against `TEST_TARGET=new` — every test for this module
2. **Full module test suite** against `TEST_TARGET=legacy` — confirm legacy still green (didn't regress while editing the harness)
3. **Type-check** the target subproject
4. **Build** the target subproject

Apply the 3-attempt fix-loop if anything fails. Stop and report if still failing after 3.

When all green, update `PROGRESS.md`: pipeline `improve` row → `✅ completed` with detail `applied X / reverted Y / total Z`, top-level **Status** → `improved`, **Current phase** → `done`. Append a history line `module improve complete: X applied, Y reverted, all tests green`.

---

## Completion report

When the module is fully improved, emit an explicit DONE marker matching `/rdd-refactor-04`'s format:

```
✅ <Seq>_<module> IMPROVED — idiomatic refactor pass complete.

  Refactors applied:  X
  Refactors reverted: Y (see improve/*.md for diagnostic notes)
  Tests on new:       Z/Z (green throughout)
  Type-check:         pass
  Build:              pass

  Intentional deviations added: <count, or "none"> (see spec/SPEC.md)

Next eligible improve target: <next_Seq>_<next_module>  (if /rdd-status reports any)
Run: /rdd-improve-05 <next_module>
(or re-run /rdd-improve-05 <module> later if more refactors are identified — improve is idempotent)
```

Next steps in the user's hands:

- Re-run `/rdd-improve-05 {module}` later if more refactors are identified
- Cutover (if not already done) — owned by the user, per `TD-08` in `TARGET.md`. Option A (big-bang) = deploy `v2/` and replace legacy. Option B (strangler-fig) = flip the feature flag for this module. Outside this pipeline either way.

---

## Anti-patterns to avoid

- **Don't refactor without the safety net.** If `/rdd-refactor-04` isn't complete, do not run this skill.
- **Don't grow refactors mid-application.** Small steps that revert cleanly are the whole point.
- **Don't disable tests to "make them pass after refactoring".** A failing test means the refactor changed behavior — revert it.
- **Don't refactor the test code while refactoring the production code.** Two separate sessions if you must.
- **Don't push high-risk refactors first.** Order by risk; build confidence.
- **Don't reintroduce confirmation prompts.** This skill is autonomous after pre-flight. Per-refactor results are logged inline; the user reviews the commit history at the end.
- **Don't escalate a failing refactor to a prompt.** Revert it after 3 attempts and continue with the next. The user sees the `reverted` list in the final report.
- **Don't run the full project suite after each refactor.** Module tests are the per-refactor signal; full suite is final verification only.

---

## When to spawn sub-agents

For long refactors (>500 lines transformed in one R-NN), spawn the appropriate code agent for the target stack with explicit parity instructions: "preserve all observable behavior; the test suite must stay green". Pass it the original code, the refactor goal, and the test files. **Verify against the suite** before accepting its output. Do not delegate the fix loop.
