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

### Phase 5 — Port (per-task loop with subagent dispatch)

**Default mode:** the main agent dispatches **one general-purpose subagent per entry point** sequentially. Same context-engineering pattern used by `/rdd-specify-03` batch mode — keeps the main agent's context bounded across long modules (10+ EPs of 500–1000 LOC each) so the work finishes without needing to `/clear` mid-flight.

The main agent's job in Phase 5 is to **dispatch, parse, and track**. The subagent's job is to **read, port, test, and report**. The contract between them is a single-line summary returned per EP.

The first time the loop runs, set the pipeline `port` row in `PROGRESS.md` to `🔄 in_progress` with detail `0/Total tasks done`. Update the row after each task completes (or fails).

#### Step 5.1 — Pick the next pending task

Read `PROGRESS.md`'s Tasks table. Find the next row with `Status: ⬜ pending` (or `🔄 in_progress` from a prior session that died mid-EP — treat as the resume target). If none → all tasks complete, proceed to Phase 6.

#### Step 5.2 — Open the task file

Create `{module_dir}/tasks/<NN>_<entry-point-name>.md` from `templates/task.md` if it doesn't exist. `NN` is the 2-digit zero-padded position from the API-surface order in `spec/SPEC.md` (`01`, `02`, ..., `99`).

Mark this task's `TaskCreate` entry `in_progress`. Update the row in `PROGRESS.md` Tasks table to `🔄 in_progress`.

#### Step 5.3 — Dispatch one general-purpose subagent

The subagent has no prior conversation context — its prompt must be fully self-contained. Use this template (substituting `<module_name>`, `<seq>`, `<NN>`, `<ep_name>`, source paths, and target paths from MAP/SPEC/scaffold):

```
You are running step "port one entry point" as part of an RDD refactor autopilot. The main agent has dispatched you with a self-contained prompt — you have no prior conversation context.

Module: <module_name>
Sequence number (Seq): <seq>
Module directory: <artifacts_dir>/<seq>_<module_name>/
Entry point: <ep_name>
Position: <NN> of <Total>
Task file: <module_dir>/tasks/<NN>_<ep_name>.md

Read first (in this order):
1. <artifacts_dir>/TARGET.md — chosen conventions, error format, multi-tenancy boundary
2. <module_dir>/spec/SPEC.md — find the API-surface row for <ep_name>; capture its BR slice, error codes, side effects
3. <module_dir>/spec/TESTS.md — find the tests planned for <ep_name>
4. <legacy_source>/<ep_legacy_path> — the legacy implementation to port
5. <target_source>/<scaffolded_stub_path> — the 501-stub created in Phase 4

Then port (parity-first):
- Fill the task file's "Spec slice" section (API surface, BRs covered, error codes, side effects)
- Port controller + service + DTO + any helpers required, mirroring legacy structure:
  - Copy legacy control flow (same if order, same early returns, same error responses, even if ugly)
  - Translate DB queries 1:1 (not "the way the new ORM prefers")
  - Preserve field names in responses
  - Preserve bugs that callers depend on; mark with `(legacy bug, intentional parity)` in BR comment
- Stay in scope: only files required by THIS entry point. Unrelated issues → record under this task's "Observations" section, do NOT act on them.

Run tests on new:
  TEST_TARGET=new <test-command for this EP's tests only — not the full suite>

Fix loop (max 3 attempts on same failing test):
- Read failure, diagnose root cause, apply focused fix, re-run THE SAME tests
- Each attempt: append one line under "Fix-loop attempts" in the task file (failure summary + diagnosis + fix)
- Never weaken tests (.skip, .only, xit, swallow errors)
- If failure points to a previously-completed EP → STOP and return FAILED (regression in committed code, main agent escalates)
- After 3 unsuccessful attempts → STOP, set task file Status: failed, return FAILED summary

Parity check against legacy:
  TEST_TARGET=legacy <test-command for this EP's tests>
  - Both must be green
  - If new green but legacy red: you fixed a legacy bug. REVERT your deviation in the new code, note in task file Observations, re-run
  - If both red on same test: adjust the test or update spec/SPEC.md BR to match what legacy actually does (parity); re-run

Improvement candidates (bugs / quirks / code smells found while reading legacy):
- Append entries to <module_dir>/spec/SPEC.md "Improvement candidates" table with class (`bug`/`quirk`/`code-smell`/`design`) + risk-if-changed note
- Do NOT change the BR — parity stays. The candidate is parallel documentation for /rdd-improve-05.

Finalize the task file:
- Status: completed
- Tests on new: X/Y, Tests on legacy: X/Y
- Source / Target file paths
- Completed timestamp
- Observations (out-of-scope findings, reverts, etc.)

Commit your work as a single commit scoped to this EP.

Forbidden:
- Do NOT use AskUserQuestion. This skill is autonomous — every decision uses the parity-first default.
- Do NOT modify another module's files (any other <NNN>_<module>/ directory).
- Do NOT modify TARGET.md, MAP.md, .rdd.yml, BATCH_SPEC.progress.md.
- Do NOT modify the upstream gate or `.rdd.yml` `skipped:` list.
- Do NOT spawn nested subagents. Delegation tree stays flat: main → 1 subagent per EP → no deeper.
- Do NOT push to remote.
- Do NOT continue to a different EP if this one fails — return FAILED summary.

Return as your final message a single one-line summary in this exact format:
Ported <ep_name> (<NN>/<Total>). Tests: X/Y on new, X/Y on legacy.

If you fail (legacy unreadable, ambiguity, fix-loop exhausted, regression in another EP), return instead:
FAILED — <ep_name>: <one-line reason>
```

#### Step 5.4 — Wait for the subagent to return

Do not dispatch the next EP until this one returns. The subagent owns the port + tests + fix-loop + commit; the main agent only tracks state.

#### Step 5.5 — Parse the summary and update state

Read the subagent's one-line summary:

- **On success** (`Ported <name> (<NN>/<Total>)...`):
  - Update the matching row in `PROGRESS.md` Tasks table → `✅ completed`
  - Increment the pipeline `port` row's detail (`X/Y tasks done`)
  - `TaskUpdate` this entry to `completed`
  - Briefly emit progress to chat (one line, exactly what the subagent returned)

- **On failure** (`FAILED — <name>: <reason>`):
  - Update the matching row in `PROGRESS.md` Tasks table → `❌ failed`
  - `TaskUpdate` this entry to `in_progress` (kept open for resumption)
  - Emit the failure to chat with the reason
  - **STOP the loop.** A failed EP is a hard fail — the user reviews and decides next steps (retry the EP after fixing the cause, adjust SPEC, or mark this EP `skipped:` in `.rdd.yml` and re-run).

#### Step 5.6 — Continue to the next pending task

If the loop is still running (no failure) and more pending tasks exist, return to step 5.1. **Do not ask the user to confirm between tasks.** The user opted into the autopilot when invoking `/rdd-refactor-04` — the only stops are hard fails.

When the last pending task completes, set pipeline `port` row in `PROGRESS.md` to `✅ completed`, append a history line `port phase complete: X/Y tasks ported`, and proceed to Phase 6.

#### Why subagent dispatch (context engineering)

- **Main agent context stays bounded** — across N EPs, main agent only accumulates: PROGRESS.md state, MAP.md (cached), TARGET.md (cached), SPEC.md (cached), and one-line summary per EP. That's a few KB of growth per EP, not 30–50k tokens (which is what inline per-EP would cost: re-read legacy + write controller/service/spec + fix-loop iterations).
- **Each subagent has fresh context** — no contamination from sibling EPs; no risk that "I already saw similar code" leads to false-pattern porting. Prompt caching warms TARGET/SPEC/TESTS, so cost per EP is predictable.
- **Failures are isolated** — EP-N failing doesn't pollute the next dispatch's context. Retry is just re-dispatch.
- **Resumability** — `PROGRESS.md` is updated after each subagent return. A session that dies mid-flight resumes on next invocation from the next pending task (Phase 0 resume check finds it).
- **Inline option** (rare): for very small EPs (<100 LOC legacy, <5 planned tests) where dispatch overhead might outweigh benefit, the main agent **may** port inline. Use sparingly — the rule of thumb is "if in doubt, dispatch". Mixed inline/dispatch within one module is fine as long as `PROGRESS.md` Tasks table reflects each task's final state.

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
- **Don't merge cutover into the refactor.** This skill produces a parity-correct module ready for cutover. The cutover mechanism is per `TD-08` in `TARGET.md` — Option A (big-bang side-by-side) deploys `v2/` wholesale at the end; Option B (strangler-fig) flips a feature flag per module. Either way, this skill stops at "module is parity-correct and tests pass" — flipping or deploying is the user's call.
- **Don't trust "looks the same" — run the suite.** Tiny differences (rounding, defaults, sort order) bite in production.
- **Don't reintroduce confirmation prompts.** This skill is autonomous after pre-flight. Hard-fail (pre-flight, fix-loop exhausted, gate blocked) is fine; "Continuar?" is not.
- **Don't lose count in the fix loop.** 3 attempts means 3 — and the count resets per test, not per phase.
- **Don't quietly edit a completed EP.** If a failure during a later EP points back to a completed one, hard-fail and surface it — that's a regression in committed code.
- **Don't push to remote.** User owns git operations beyond local commits.
- **Don't bypass the upstream-port gate.** It exists to catch migration-order mistakes that prose-only `MAP.md` cannot prevent. If a real upstream genuinely isn't going to be ported, mark it `skipped:` in `.rdd.yml` with a reason — never edit Phase 0 to ignore it, and never fabricate a fake `PROGRESS.md` to make the gate think a module is done.
- **Don't escalate ambiguity to a prompt** — apply the documented parity-first default and continue. Surfacing a question every time legacy and new disagree defeats autonomy. If the parity default is wrong for a specific case, the user reverts the commit and edits `SPEC.md` to record the deviation; then re-runs the skill.
- **Don't ask "fix the bug or keep parity?".** Migration is parity by definition. Bugs are recorded in `spec/SPEC.md` "Improvement candidates" as input for `/rdd-improve-05`. Asking the user to choose at port time defeats the entire pipeline — the answer is *always* parity here, the fix discussion happens in a separate session.

---

## Sub-agent dispatch summary

Phase 5 of this skill **defaults to subagent-per-EP dispatch** (see "Phase 5 — Port" above). The main agent dispatches one general-purpose subagent per entry point, tracking state in `PROGRESS.md` and the Tasks table; the subagent does the read + port + tests + fix-loop + commit and returns a one-line summary.

- **Default:** every EP gets its own subagent. Delegation tree is flat (main → 1 subagent per EP → no deeper).
- **Inline option** (rare): for very small EPs (<100 LOC legacy, <5 planned tests), main agent may port inline. Mixed inline/dispatch within one module is fine.
- **Other phases** (Phase 0 preflight, Phase 1 plan_tests, Phase 2 harness, Phase 3 lock_legacy, Phase 4 scaffold, Phase 6 side_effects, Phase 7 final_verification): main agent runs inline. These phases either span the whole module (planning, scaffold) or cross multiple EPs (full-suite verification) — subagent dispatch doesn't fit.
- **Do not delegate Phase 6/7** — the main agent must see the full suite results to detect cross-EP regressions; a subagent with fresh context wouldn't have that visibility.
