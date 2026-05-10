---
name: rdd-refactor-04
description: Use this skill after rdd-specify-03 to plan characterization tests, lock legacy behavior, and port the module to the target stack with parity. Fourth step in the RDD pipeline (per-module). Combines test planning + implementation + port — TDD and refactoring are inseparable phases of the same act. Triggers on "refactor X", "port X", "rewrite X with parity", "implement the migration of X". Produces spec/TESTS.md, working code, tasks/NN_&lt;name&gt;.md per entry point, updates the module's PROGRESS.md.
---

# rdd-refactor — Plan tests, lock legacy, port with parity

You are executing the rewrite of one module. This is the only RDD skill that produces runnable code. The skill merges what would otherwise be a redundant handoff between "plan tests" and "implement port" — both are part of the same refactoring act.

You must:

1. **Plan characterization tests** from `SPEC.md` → `TESTS.md`
2. **Set up the test harness**
3. **Implement the tests against the legacy** until they pass — proves they capture current behavior
4. **Scaffold the new module** in the target stack
5. **Port the module** entry point by entry point with parity (parity-first, not idiomatic-first)
6. **Verify side effects**
7. **Run final verification** (full suite + type-check + build)
8. **Stop.** Idiomatic improvement is `/rdd-improve-05` — a separate session.

This skill runs **autonomously**. The legacy code, `SPEC.md`, `TARGET.md`, and `TESTS.md` are the source of truth — there are no per-phase confirmation prompts. The skill stops only when it genuinely cannot continue:

- **Pre-flight fails** (missing inputs, blocked upstream gate, dirty branch) — hard fail, surface error, exit
- **Fix-loop exhausts 3 attempts** on the same test — hard fail, surface failure output and root-cause hypothesis, exit
- **Final verification fails after 3 fix attempts** — hard fail, exit

Ambiguous mid-port situations resolve via parity-first defaults (documented per-step below). The user reviews the resulting commits and can intervene by editing `SPEC.md` and re-running, or by `git revert` / `git reset` if a port direction was wrong. No `Continuar?` prompts, no mid-flight confirmations — the user invoked the skill, the skill executes.

---

## Inputs

The user invokes the skill with a module name (e.g., "/rdd-refactor-04 products"). Resolve:

- `{artifacts_dir}/TARGET.md` — must exist (chosen test framework, test posture, error format, conventions)
- `{artifacts_dir}/MAP.md` — must exist; look up the module's **`Seq`** (sequence number) in the Modules table
- `{module_dir}/spec/SPEC.md` — must exist with BRs and Error Catalog filled. `module_dir` is `{artifacts_dir}/<Seq>_<module>/` (e.g., `rdd/003_products/`); the user can invoke with either the bare name (`products`) or the prefixed form (`003_products`).
- `{module_dir}/PROGRESS.md` — must exist (created by `/rdd-specify-03`). Aggregate state for the module; this skill updates the pipeline table, the tasks table, and the upstream gate during execution.

If any is missing, instruct the user to run the upstream skill first. **Do not proceed.**

`spec/TESTS.md` is **produced by this skill** in Phase 1 — don't expect it as input.

### Module directory resolution

Same three-form input as `/rdd-specify-03`: bare name (`products`), prefixed (`003_products`), or Seq only (`003`). Resolution:

1. Parse the input:
   - `^[0-9]{3}$` → Seq-only; look up MAP for the row with this Seq, take its name.
   - `^[0-9]{3}_(.+)$` → prefixed; split into Seq and name.
   - Otherwise → bare name; look up MAP for the row with this name, take its Seq.
2. Glob `{artifacts_dir}/[0-9][0-9][0-9]_<module>/`. Exactly one match expected — the directory was created by `/rdd-specify-03`. If no match, **stop**: spec hasn't been written, run `/rdd-specify-03 <module>` first.
3. The resolved path is `module_dir`. All references below to `spec/SPEC.md`, `spec/TESTS.md`, `tasks/NN_<name>.md`, `PROGRESS.md` are relative to `module_dir`.
4. **Backward compatibility:** if MAP.md has no `Seq` column and only a bare-name flat directory exists (with `SPEC.md` at the root, no `spec/` subdir), accept it and proceed — but emit a one-line warning suggesting the user re-run `/rdd-map-codebase-02` and migrate to the new layout (`SPEC.md → spec/SPEC.md`, create `PROGRESS.md`, etc).

---

## Module layout (this skill writes here)

```
{module_dir}/
├── PROGRESS.md          ← updated continuously: pipeline rows, task table, upstream gate, history
├── spec/
│   ├── SPEC.md          ← input (read-only here; written by /rdd-specify-03)
│   └── TESTS.md         ← Phase 1 output (test plan)
└── tasks/               ← Phase 5 output (one file per entry point being ported)
    ├── 01_<entry-point-name>.md
    ├── 02_<entry-point-name>.md
    └── ...
```

`PROGRESS.md` is the **single source of truth** for `/rdd-status` and for resuming this skill mid-flight. Per-task files (`tasks/NN_<name>.md`) hold the detailed per-entry-point notes (test counts, fix-loop attempts, observations, source/target paths). The pipeline table in `PROGRESS.md` reflects the high-level phase state; the tasks table mirrors per-task status from those files.

**Update discipline:** every phase transition writes to `PROGRESS.md` (update the pipeline row + append a one-line history entry with timestamp). Every per-task transition writes to `tasks/NN_<name>.md` and updates the matching row in `PROGRESS.md`'s tasks table. The two files stay consistent — never update one without the other.

---

## Procedure

### Phase 0 — Preflight

Before designing tests:

1. **Inputs present.** `TARGET.md`, `MAP.md`, `{module_dir}/spec/SPEC.md`, and `{module_dir}/PROGRESS.md` exist. `SPEC.md` Business Rules section is non-empty; every BR has a source citation; Error Catalog codes referenced by BRs exist in the catalog. Open Questions in `SPEC.md` are resolved (or have a recommended default that the skill will adopt as parity).
2. **Upstream-port gate (HARD BLOCK).** This is the gate that enforces migration order. Read `MAP.md`, get this module's `Seq` (call it `current_seq`). For every directory matching `{artifacts_dir}/[0-9][0-9][0-9]_*/` with a 3-digit prefix `< current_seq`:
   - **OK** — the directory has `PROGRESS.md` with the `port` row in the Pipeline table marked `✅ completed` (or the legacy `REFACTOR.progress.md` showing `Status: completed`, for backward compat).
   - **OK** — `.rdd.yml` declares the module as `skipped:` (see schema below).
   - **NOT OK** — neither of the above. The module hasn't been ported and isn't explicitly skipped.

   If any upstream is NOT OK, **STOP**. Emit a structured error:

   ```
   Cannot start /rdd-refactor-04 <Seq>_<module> — upstream ports incomplete.

   Upstream modules (Seq < <current_seq>) without completed port:
     - <Seq>_<upstream-1>  (PROGRESS.md: port row is <pending | in_progress | not found>)
     - <Seq>_<upstream-2>  (PROGRESS.md: not found — module never specced)

   Resolve in one of two ways:
     a) Complete the port: run /rdd-refactor-04 <Seq>_<upstream-N> for each, in Seq order
     b) Skip the upstream (won't be ported in this migration): add to .rdd.yml under `skipped:`
        Example:
            skipped:
              - module: <upstream-name>
                reason: deprecated; will be retired post-cutover

   Re-run /rdd-refactor-04 <module> after every upstream is either completed or marked skipped.
   ```

   This gate exists to catch ordering mistakes the MAP cannot prevent (e.g., starting a Wave 4 transactional module before a Wave 2 dependency is ready). The user can override per-module by marking specific upstreams as `skipped:` in `.rdd.yml` — never bypass the gate silently.

   **Skip schema for `.rdd.yml`:**

   ```yaml
   skipped:
     - module: agent-tutorial-helper       # bare module name (no Seq prefix)
       reason: deprecated; not in production use
     - module: legacy-batch-job
       reason: replaced by scheduled task in foundation; no port needed
   ```

   Each entry must include both `module` and a non-empty `reason` (the reason becomes the audit trail when someone later asks "why did we skip X?").

3. **Branch check.** `git status` and `git branch --show-current`. If on `main` or default, or uncommitted changes touching files outside the migration scope, **stop** and instruct the user to set up the right branch (this is a hard fail — the skill cannot safely commit).
4. **Target subproject readiness.** `target.source` directory exists, dependencies installed.
5. **Resume check.** Read `PROGRESS.md`. The Pipeline table shows which phases are ✅/🔄/⬜; the Tasks table shows per-task status. Print one line summarizing where the skill picks up: *"Resuming from phase=`port`, X/Y tasks complete. Next task: `NN_<name>`."* Then proceed — no confirmation needed.

If preflight passes, update `PROGRESS.md` (mark the upstream gate satisfied, set `preflight` row → ✅, append history line). Create the persistent task list (`TaskCreate`) — one entry per pipeline phase plus one per entry point in the SPEC's API surface:

- Plan tests
- Set up harness
- Lock legacy
- Scaffold
- Port: <entry-point-name-1>
- Port: <entry-point-name-2>
- ...
- Verify side effects
- Final verification

Use the entry point's real name (no `EP-N` prefix in user-facing tasks). The numeric position becomes the file prefix in `tasks/` (`01_<name>.md`, `02_<name>.md`...).

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

Write the test plan to `{module_dir}/spec/TESTS.md`. Update `PROGRESS.md`: pipeline `tests` row → `✅ completed` with counts (`N tests, M rejected`); append history line. Print one line and proceed to Phase 2:

```
Tests planned for {module}: N tests covering Y BRs (rejected: M). Path: {module_dir}/spec/TESTS.md
```

### Phase 2 — Set up the test harness

Per the harness plan from Phase 1:

- Install/configure the test framework (`target.test_framework`)
- Wire up the test database
- Build the dual-target client abstraction
- Configure auth, time, random

Commit this as a discrete, reviewable change.

Update `PROGRESS.md`: pipeline `harness` row → `✅ completed`; append history line. Proceed directly to Phase 3.

### Phase 3 — Lock legacy behavior

Implement every test from `TESTS.md` pointing at the **legacy** implementation. Run them. **Expect them to pass.**

If a test fails, resolve **autonomously** with parity-first defaults — do not stop to ask. Migration locks legacy behavior verbatim; bugs and quirks are the rule, not exceptions.

- **Test asserts behavior the legacy doesn't have** (test was over-specific) → adjust the test to match what legacy actually does. Append a history line to `PROGRESS.md` summarizing the adjustment (the actual change is committed to the test file).
- **Spec describes behavior the legacy doesn't have** (BR was wrong) → update the BR in `spec/SPEC.md` to match legacy, mark it `(legacy behavior verified during lock; spec corrected)`. Adjust the test accordingly.
- **Legacy has a bug or quirk not in spec** → update `spec/SPEC.md` BR to match the bug verbatim, mark `(legacy bug, intentional parity)`. Adjust the test. **Also append the bug as an entry to `spec/SPEC.md` "Improvement candidates"** with a class label (`bug` / `quirk`) and a one-line risk-if-changed note — that's the input for `/rdd-improve-05` later. Never re-litigate "fix or keep" here; parity is the answer for this skill, improvement candidates are the parallel record for the next skill.

Re-run after each adjustment. Apply the standard 3-attempt fix-loop discipline per test (read failure, diagnose, focused fix, re-run). After 3 unsuccessful attempts on the same test, **stop** with a hard-fail report — the situation is genuinely ambiguous and needs human judgment.

Never adjust legacy code to make tests pass. Legacy is the source of truth at this stage.

When all tests are green against legacy, commit. **This is the behavior lock.**

Update `PROGRESS.md`: pipeline `lock_legacy` row → `✅ completed` with detail (`X/Y on legacy`). If spec/test adjustments were made during this phase, append a history line summarizing them (the actual changes are already committed to `spec/SPEC.md` / test files). Proceed directly to Phase 4.

### Phase 4 — Scaffold the new module

Create the module under `target.source` per `TARGET.md` conventions:

- Module/package directory
- Entry points matching the spec's API surface (one route per spec endpoint)
- Empty implementations returning 501 or throwing "not implemented"
- Required DI / module registration

Tests against the new target should fail cleanly.

Commit. Update `PROGRESS.md`: pipeline `scaffold` row → `✅ completed`; append history line. Initialize the Tasks table in `PROGRESS.md` with one row per entry point from `spec/SPEC.md`'s API surface (status `⬜ pending`, file `tasks/NN_<name>.md` not yet created). Proceed directly to Phase 5.

### Phase 5 — Port (per-task loop)

For each entry point in order (read from `spec/SPEC.md` API surface), execute these steps. **Do not batch.** Each entry point is one task — one file under `tasks/`, one TaskCreate row, one commit when complete.

The first time the loop runs, set the pipeline `port` row in `PROGRESS.md` to `🔄 in_progress` with detail `0/Total tasks done`. Update the row after each task completes.

#### Step 5.1 — Open the task

Create `{module_dir}/tasks/NN_<entry-point-name>.md` from `templates/task.md` if it doesn't exist (resume case: file may already exist from a prior session — re-open it with `Status: in_progress`). `NN` is the 2-digit zero-padded position from the API-surface order in `SPEC.md` (`01`, `02`, ..., `99`).

Re-read the entry point's section in `spec/SPEC.md` and the corresponding tests in `spec/TESTS.md`. Internal checklist: one item per BR + one per side effect + "run tests on new". Fill in the `Spec slice` section of the task file (API surface, BRs covered, error codes, side effects). Mark this task's `TaskCreate` entry `in_progress`. Update `PROGRESS.md`'s tasks table row to `🔄 in_progress`.

#### Step 5.2 — Port the logic (parity-first)

- **Copy the structure** from legacy, adapting only for syntax differences (Deno → Node, JS → TS, framework idioms required to compile)
- **Keep the same control flow.** Same `if` order, same early returns, same error responses. Even if it looks ugly.
- **Keep the same DB queries.** Translate 1:1, not "the way the new ORM prefers"
- **Keep the same field names** in responses
- **Keep bugs that callers depend on** (`legacy bug, intentional parity`)

Stay in scope: only files required by **this entry point**. Unrelated issues → record in this task file's `Observations` section. **Do not act on them.** If reading legacy reveals a bug or quirk worth flagging for `/rdd-improve-05`, append it to `spec/SPEC.md` "Improvement candidates" (class + risk note) and continue porting verbatim — the port honors legacy, the candidate is the record for later.

#### Step 5.3 — Run only this task's tests against new

```
TEST_TARGET=new <test-command for this entry point's tests>
```

Not the full suite. Save that for final verification.

#### Step 5.4 — Fix loop (max 3 attempts)

If green, proceed to step 5.5.

If red, enter the fix loop. Read failure output, diagnose root cause, apply focused fix, re-run **the same tests**. Maximum **3 attempts**. Count deliberately. Each attempt appends a one-line entry under `Fix-loop attempts` in this task file (with the failure summary, root-cause hypothesis, and what you changed).

**Discipline:**
- **Read the error.** Same fix applied twice = wrong diagnosis.
- **Fix the root cause.** Don't weaken tests. Don't add `.skip`, `.only`, `xit`. Don't catch and swallow errors.
- **Stay in scope.** Failure pointing to a previously-completed task → **stop**, escalate (regression in committed code).
- **No shortcuts.** Never disable hooks. Never bypass safety checks.

After 3 unsuccessful attempts, **stop**. Update the task file's status to `failed`, mirror to `PROGRESS.md` tasks table, then report: the entry point name (not `EP-N`), concise failure output, root-cause hypothesis, attempts made. Wait for user.

#### Step 5.5 — Run the same tests against legacy (parity check)

```
TEST_TARGET=legacy <test-command for this EP's tests>
```

Both must be green. If a test passes on `new` but fails on `legacy`, you accidentally fixed a legacy bug. **Default: revert** the deviation in the new code so behavior matches legacy (parity-first). Note the revert in this task file's `Observations` section so the user can review if they want to deliberately deviate later. Do not stop to ask — parity is the rule, deviations are an explicit user decision recorded in `spec/SPEC.md` under "Intentional deviations from legacy".

If both targets fail the same test (test asserts behavior neither implements), apply the lock_legacy resolution rules: adjust the test or update `spec/SPEC.md` to match the actual legacy behavior, then re-run.

#### Step 5.6 — Close the task and continue

Finalize the task file: set `Status: completed`, record final test counts (new + legacy), `Completed:` timestamp, observations. Mirror the row in `PROGRESS.md`'s tasks table to `✅ completed`. Increment the pipeline `port` row's detail (`X/Y tasks done`). `TaskUpdate` this entry to `completed`.

Print one progress line using the **entry point's real name** and **proceed directly to step 5.1 of the next task**:

```
Ported <entry-point-name> (N/Total). Tests: X/Y on new, X/Y on legacy.
```

Example: `Ported agent-relatorios (1/5). Tests: 12/12 on new, 12/12 on legacy.`

When the last task is done, set pipeline `port` row in `PROGRESS.md` to `✅ completed`, append a history line, and proceed to Phase 6.

### Phase 6 — Verify side effects

Run all side-effect tests against `TEST_TARGET=new`. Pay extra attention:

- **Webhook payloads** — must match byte-for-byte (modulo timestamps, ids)
- **Email/notification content** — must match
- **Audit log entries** — must match shape
- **Queue messages** — must match shape and routing key

If a side-effect mismatch is unrecoverable after the standard 3-attempt fix-loop, surface it under `Intentional deviations from legacy` in `spec/SPEC.md` (don't stop) — the user reviews the deviation when they review the commits. If the mismatch can be fixed, fix it.

Update `PROGRESS.md`: pipeline `side_effects` row → `✅ completed`; append history line. Proceed directly to Phase 7.

### Phase 7 — Final verification

Run:

1. **Full module test suite** against `TEST_TARGET=new`
2. **Full module test suite** against `TEST_TARGET=legacy` (confirm legacy didn't regress while editing the harness)
3. **Type-check** the target subproject
4. **Build** the target subproject

Apply the 3-attempt fix-loop if anything fails. Stop and report if still failing after 3.

When all green, update `PROGRESS.md`: pipeline `final` row → `✅ completed`, top-level **Status** → `port_complete`, **Current phase** → `done`. Append a history line `module port complete: X/Y tasks, all suites green`.

### Phase 8 — Stop. Do not refactor for style yet.

This is the hardest discipline step. The new code is parity-correct but probably not idiomatic. **Stop anyway.**

Idiomatic refactoring is `/rdd-improve-05` — a separate session. The tests from this skill guard that refactor. That's their second job.

---

## Completion report

When the module is fully refactored, emit an **explicit DONE marker** so the user knows unambiguously that the module is finished (not just a phase). Compute the next eligible module by reading `MAP.md` for the lowest `Seq` greater than this module's, and check that its upstreams are all completed/skipped (same logic as `/rdd-status`'s "Next eligible").

Format:

```
✅ <Seq>_<module> DONE — port + final verification complete.

  Tasks:           X/Y completed
  Tests on new:    X/Y
  Tests on legacy: X/Y
  Type-check:      pass
  Build:           pass

  Intentional deviations: <count, or "none"> (see spec/SPEC.md)
  Out-of-scope observations: <count> (see tasks/*.md Observations sections)

Next eligible module: <next_Seq>_<next_module>
Run: /rdd-refactor-04 <next_module>
(or /rdd-status to see the full board, or /rdd-improve-05 <module> to refactor this one for style now that parity is locked)
```

If there is no next eligible module (every higher-`Seq` module is already done or skipped), emit instead:

```
🎉 All modules in MAP are now ported. Migration porting phase complete.
Run /rdd-status for the final board, or /rdd-improve-05 on any module to start the idiomatic-quality pass.
```

This is the **one user-facing stop** at the end of the skill — not a confirmation prompt, but a clearly marked end state. The next session picks up from `/rdd-refactor-04 <next_module>` knowing exactly where it lands.

Git operations beyond local commits (push, PR) are out of scope — the user owns version control. You commit at the boundaries specified above but do not push.

---

## Anti-patterns to avoid

- **Don't refactor while porting.** "Improving while you're already there" turns a parity port into a behavior change. Phase 8 stop is non-negotiable.
- **Don't translate to "idiomatic" early.** Even if legacy is bad, copy shape-by-shape. Clean up after parity is proved.
- **Don't skip the legacy-side test run** (Phase 3 and step 5.5). Tests never validated against legacy might pass on new for the wrong reason.
- **Don't merge cutover into the refactor.** This skill produces a parity-correct module behind a feature flag. Flipping the flag is separate.
- **Don't trust "looks the same" — run the suite.** Tiny differences (rounding, defaults, sort order) bite in production.
- **Don't reintroduce confirmation prompts.** This skill is autonomous after pre-flight. Hard-fail (pre-flight, fix-loop exhausted, gate blocked) is fine; "Continuar?" is not.
- **Don't lose count in the fix loop.** 3 attempts means 3 — and the count resets per test, not per phase.
- **Don't quietly edit a completed EP.** If a failure during a later EP points back to a completed one, hard-fail and surface it — that's a regression in committed code.
- **Don't push to remote.** User owns git operations beyond local commits.
- **Don't bypass the upstream-port gate.** It exists to catch migration-order mistakes that prose-only `MAP.md` cannot prevent. If a real upstream genuinely isn't going to be ported, mark it `skipped:` in `.rdd.yml` with a reason — never edit Phase 0 to ignore it, and never fabricate a fake `PROGRESS.md` to make the gate think a module is done.
- **Don't escalate ambiguity to a prompt** — apply the documented parity-first default and continue. Surfacing a question every time legacy and new disagree defeats autonomy. If the parity default is wrong for a specific case, the user reverts the commit and edits `SPEC.md` to record the deviation; then re-runs the skill.
- **Don't ask "fix the bug or keep parity?".** Migration is parity by definition. Bugs are recorded in `spec/SPEC.md` "Improvement candidates" as input for `/rdd-improve-05`. Asking the user to choose at port time defeats the entire pipeline — the answer is *always* parity here, the fix discussion happens in a separate session.

---

## When to spawn sub-agents

For long porting tasks (>1000 lines per EP), spawn the appropriate code-writing agent for the target stack with explicit instructions: "parity-first, copy structure, do not refactor". Pass it the legacy file + spec section. **Verify against the test suite** before accepting. Do not delegate the fix loop — run that yourself with full context.
