---
name: rdd-status
description: Use this skill anytime to see migration progress across all modules and phases. Reads each module's PROGRESS.md (and falls back to legacy artifacts for older projects) and prints a clean phase-by-phase status table. Triggers on "where are we", "migration status", "what's done", "what's left", "show progress".
---

# rdd-status — Show migration progress

You are reporting RDD migration progress. **You generate no new artifacts.** You read existing files in `{artifacts_dir}` and synthesize state. The artifacts on disk are the source of truth — there is no separate progress database to maintain or sync.

The primary signal per module is `{module_dir}/PROGRESS.md` (written by `/rdd-specify-03`, `/rdd-refactor-04`, `/rdd-improve-05`). It already aggregates pipeline phase, task status, upstream gate, and history — your job is to tabulate across modules, not to re-derive state.

## Procedure

### 1. Load configuration

Read `.rdd.yml` for `artifacts_dir`. If absent, tell the user RDD isn't configured yet — point at `/rdd-specify-01` to get started.

### 2. Detect setup phase status

Check `{artifacts_dir}/`:

- **TARGET.md**
  - Doesn't exist → `target: not started`
  - Exists, Decisions Summary has any `<pending>` → `target: in progress (X/Y TDs decided)`
  - Exists, all decisions filled → `target: completed`
  - If `.rdd.yml` has `skip_target: true` → `target: skipped (in-place mode)`
- **MAP.md**
  - Doesn't exist → `map: not started`
  - Exists → `map: completed (N modules identified)`
- **BATCH_SPEC.progress.md** (only if a batch spec autopilot is or was running)
  - Doesn't exist → don't show this row
  - Exists, `Status: in_progress` → `batch_spec: in progress (M/N modules done, K failed)` — also note next pending module so user knows where to resume
  - Exists, `Status: completed` → `batch_spec: completed (M/N modules done, K failed)`

### 3. Detect per-module status

List all module directories under `{artifacts_dir}/` matching `[0-9][0-9][0-9]_*/` (sequence-prefixed, the standard layout). **Sort ascending by the 3-digit prefix** — that's the migration order. For each:

**Primary path — read `{module_dir}/PROGRESS.md`:**

The file's Pipeline table tells you the status of each phase (`⬜ pending` / `🔄 in_progress` / `✅ completed` / `⏭ skipped`). Map to columns directly:

| Column   | Source in PROGRESS.md                              |
|----------|----------------------------------------------------|
| spec     | Pipeline row `spec`                                |
| tests    | Pipeline row `tests`                               |
| port     | Pipeline row `port` + detail (X/Y tasks done)      |
| improve  | Pipeline row `improve` + detail (applied/reverted) |

The top-level `Status:` field tells you the module's overall state: `not_started` / `in_progress` / `port_complete` / `improved` / `done`. Use this for the legend column.

**Fallback path — legacy layout (no PROGRESS.md):**

If a module dir has no `PROGRESS.md` but has the old artifacts (`SPEC.md` at root, `REFACTOR.progress.md`, `IMPROVE.progress.md`), fall back to artifact-by-artifact inference (see appendix below) and emit a one-line note: *"`<Seq>_<module>` uses legacy layout — re-run `/rdd-specify-03 <module>` to migrate to PROGRESS.md."* Don't refuse to report; just tell the user how to upgrade.

**Modules in MAP without a directory:** list with all columns `⬜ not started`, ordered by their `Seq` from MAP.

**Modules in `.rdd.yml`'s `skipped:`:** show `port: ⏭ skipped (<reason>)`. Skipped modules satisfy upstream gates for downstream modules (per `/rdd-refactor-04` Phase 0 rules).

**Backward compatibility — no Seq prefix anywhere:** if no directories match `[0-9][0-9][0-9]_*/` but bare-name dirs exist (legacy layout from a prior plugin version), fall back to alphabetical sort by directory name and emit a one-line header note: *"Note: legacy layout detected (no Seq prefix). Re-run `/rdd-map-codebase-02` to assign sequence numbers, then `git mv <module> <Seq>_<module>` and migrate each to the new spec/tasks/improve layout via re-running `/rdd-specify-03 <module>`."*

### 4. Detect "currently active"

Currently active work exists when any of these is true:

- `BATCH_SPEC.progress.md` shows `Status: in_progress` → autopilot interrupted; resume with `/rdd-specify-03` (no arg)
- A module's `PROGRESS.md` has top-level `Status: in_progress` and pipeline `port` row in `🔄 in_progress` → port paused; resume with `/rdd-refactor-04 <module>`. The Tasks table shows which task is the current one (`🔄 in_progress`).
- A module's `PROGRESS.md` has pipeline `improve` row in `🔄 in_progress` → improve paused; resume with `/rdd-improve-05 <module>`. The Improve refactors table shows the current refactor.

Highlight these — they're where work resumes. If batch_spec is in progress AND a module's port is in progress, both are listed (the user is doing both phases on different modules).

If multiple modules' `PROGRESS.md` are simultaneously `in_progress` on port or improve, that's a flag — typically the user should finish one before starting another. Note it but don't enforce it.

### 5. Detect blockers

Look for signs of blocked work:

- A `tasks/NN_<name>.md` file with `Status: failed` and 3 fix-loop attempts logged → port stuck on a test failure for that entry point
- An `improve/NN_<name>.md` file with `Status: reverted` → that refactor couldn't be applied; user can hand-craft in a separate session
- Open questions in `spec/SPEC.md` still unresolved after `spec/TESTS.md` exists → tests planned without full spec
- `BATCH_SPEC.progress.md` has any modules with `Status: failed` → list them; user must retry individually
- `MAP.md` lists modules not yet started after a long elapsed time (compare file timestamps if signal is weak)

### 5.5. Compute "next eligible" module

Given the sequence-ordered list and per-module status, find the **next eligible module** for `/rdd-refactor-04`:

1. Walk modules in ascending `Seq` order.
2. Skip modules with pipeline `port` row `✅ completed` and modules listed in `.rdd.yml` `skipped:`.
3. For the first module with port row `⬜ pending` or `🔄 in_progress`:
   - Check every upstream (every module with lower `Seq`): is its port row `✅ completed` OR listed in `.rdd.yml` `skipped:`?
   - **If yes for all** — this is the next eligible. Stop walking.
   - **If no for some** — note which upstreams block it; continue walking (a later module might be eligible if all its upstreams happen to be completed/skipped).
4. If a module is currently `🔄 in_progress` (active work paused), surface that as the resume target instead — it takes priority over picking a new "next" module.

Read `.rdd.yml` once at the start of this step to capture the `skipped:` list (bare module names with reasons).

Same logic, separately, for `/rdd-improve-05`: the next eligible module to *improve* is the lowest-`Seq` module whose port row is `✅ completed` AND whose improve row is `⬜ pending` AND whose upstreams are all `port: completed/skipped`.

### 6. Emit the report

Format the output as a single, scannable response. Suggested structure:

```
RDD Status — <project root>
Artifacts: <artifacts_dir>

Setup
─────
target:  ✅ completed (8/8 TDs decided)
map:     ✅ completed (12 modules identified)

Modules (sorted by Seq)                                                 (legend at bottom)
───────────────────────                                                 ┌─────────────────────────────────┐
                       spec    tests   port              improve   wave │ ⬜ not started                   │
005_foundation         ✅      ✅      ✅                 ✅        0    │ 🔄 in progress (with detail)    │
010_products           ✅      ✅      ✅                 ✅        2    │ ✅ completed                     │
015_customers          ✅      ✅      🔄 EP 4/8         ⬜        2    │ ⏭ skipped (.rdd.yml)            │
020_sales              ✅      ⬜      ⬜                 ⬜        4    └─────────────────────────────────┘
025_appointments       🔄 (open questions)  ⬜  ⬜       ⬜        3
030_commissions        ⬜      ⬜      ⬜                 ⬜        4
... (remaining modules from MAP)

Currently active
────────────────
• 015_customers — /rdd-refactor-04 paused at EP-4 of 8. Resume: /rdd-refactor-04 customers

Next eligible
─────────────
• 015_customers (resume in progress) — once complete, 020_sales is next (upstreams 005, 010, 015 all completed/skipped)

Blockers
────────
• None

Suggested next step
───────────────────
Resume /rdd-refactor-04 015_customers, or start /rdd-specify-03 on a higher-Seq module if you want to parallelize spec work ahead. Phase 0 of /rdd-refactor-04 will block any module whose upstreams aren't completed.
```

Adjust the layout to fit whatever's actually in the artifacts. The goal is **at-a-glance comprehension**: the user looks at this once and knows where they are.

### 7. Respect missing inputs

If `.rdd.yml` doesn't exist or `artifacts_dir` is empty, don't fail — just say "no RDD work has started yet" and point at `/rdd-specify-01`.

If MAP.md doesn't exist but per-module dirs do (rare — user manually created), report what you find with a note about the missing MAP.

## Anti-patterns to avoid

- **Don't invent state.** If `PROGRESS.md` doesn't exist, don't assume the phase is complete or in progress — report `not started` for that module (or fall back to legacy inference if old artifacts exist).
- **Don't write any files.** Status is read-only. `PROGRESS.md` is written by `/rdd-specify-03`, `/rdd-refactor-04`, `/rdd-improve-05` — never by this skill. No caches, no derived state files.
- **Don't be silent on partial progress.** A spec with open questions is half-done, not done — say so.
- **Don't suppress blockers.** Failed tasks (`tasks/NN_<name>.md` `Status: failed`), reverted refactors, contradictions between PROGRESS.md and the per-task files — surface them.
- **Don't sort modules by status.** Sort by the `Seq` prefix of the directory (ascending). That's the migration order — mixing it hides the shape.
- **Don't bloat the report.** A user runs this skill multiple times a day — keep it dense and scannable.
- **Don't compute "next eligible" without consulting `.rdd.yml`'s `skipped:` list.** A module with a skipped upstream is still eligible — silently treating skipped-upstreams as blockers makes the recommendation useless.

---

## Appendix — Legacy artifact inference (fallback only)

For modules without `PROGRESS.md` (older projects pre-migration), infer state from the old per-artifact layout:

| Artifact (legacy)                                | Inference                                    |
|--------------------------------------------------|----------------------------------------------|
| no `SPEC.md` at module root                      | `spec: not started`                          |
| `SPEC.md` exists, open questions flagged         | `spec: in progress (open questions)`         |
| `SPEC.md` exists, no open questions              | `spec: completed (N BRs)`                    |
| no `TESTS.md`                                    | `tests: not started`                         |
| `TESTS.md` exists                                | `tests: completed (N tests planned)`         |
| no `REFACTOR.progress.md`                        | `port: not started`                          |
| `REFACTOR.progress.md` with `Status: in_progress`| `port: in progress (phase=X, EP Y/Z)`        |
| `REFACTOR.progress.md` with `Status: completed`  | `port: completed (Y/Z tests passing)`        |
| no `IMPROVE.progress.md`                         | `improve: not started` (or `n/a` if port ✗)  |
| `IMPROVE.progress.md` with `Status: in_progress` | `improve: in progress (refactor X/Y)`        |
| `IMPROVE.progress.md` with `Status: completed`   | `improve: completed`                         |

Always emit the per-module migration hint when this fallback fires.

## Bonus: quick-look mode

If the user says "/rdd-status quick" or "/rdd-status summary", return only the high-level counts:

```
Setup: target ✅  map ✅
Modules: 3/12 done  •  1 in progress  •  8 not started
Currently active: 015_customers (port, EP 4/8)
Next eligible: 015_customers (in progress)  →  020_sales after
Blockers: none
```

That's enough for "where am I now" without scanning the full table.
