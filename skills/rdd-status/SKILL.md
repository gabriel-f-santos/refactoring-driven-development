---
name: rdd-status
description: Use this skill anytime to see migration progress across all modules and phases. Reads existing RDD artifacts (TARGET.md, MAP.md, per-module SPEC/TESTS/PORT.progress/IMPROVE.progress files) and prints a clean phase-by-phase status table. Triggers on "where are we", "migration status", "what's done", "what's left", "show progress".
---

# rdd-status — Show migration progress

You are reporting RDD migration progress. **You generate no new artifacts.** You read existing files in `{artifacts_dir}` and infer state. The artifacts on disk are the source of truth — there is no separate progress database to maintain or sync.

## Procedure

### 1. Load configuration

Read `.rdd.yml` for `artifacts_dir`. If absent, tell the user RDD isn't configured yet — point at `/rdd-target` to get started.

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

### 3. Detect per-module status

For each module dir under `{artifacts_dir}/` (excluding `.` and skipping non-directories), check the artifacts:

| Artifact | Inference |
|----------|-----------|
| no `SPEC.md` | `spec: not started` |
| `SPEC.md` exists, has open questions still flagged | `spec: in progress (open questions)` |
| `SPEC.md` exists, no open questions | `spec: completed (N BRs)` |
| no `TESTS.md` | `tests: not started` |
| `TESTS.md` exists | `tests: completed (N tests planned)` |
| no `PORT.progress.md` | `port: not started` |
| `PORT.progress.md` with `Status: in_progress` | `port: in progress (phase=X, EP Y/Z)` |
| `PORT.progress.md` with `Status: completed` | `port: completed (Y/Z tests passing)` |
| no `IMPROVE.progress.md` | `improve: not started` (or `n/a` if port isn't done) |
| `IMPROVE.progress.md` with `Status: in_progress` | `improve: in progress (refactor X/Y)` |
| `IMPROVE.progress.md` with `Status: completed` | `improve: completed` |

Also cross-reference with `MAP.md` — list any module that's in `MAP.md` but has no directory yet (those haven't been started).

### 4. Detect "currently active"

A module is "currently active" if any of its progress files (`PORT.progress.md`, `IMPROVE.progress.md`) has `Status: in_progress`. Highlight these — they're where work resumes.

If multiple modules are simultaneously in progress, that's a flag — typically the user should finish one before starting another. Note it but don't enforce it.

### 5. Detect blockers

Look for signs of blocked work:

- `PORT.progress.md` shows fix-loop attempts at 3 with no completion → port is stuck on a test failure
- Open questions in `SPEC.md` still unresolved after `TESTS.md` exists → tests planned without full spec
- `MAP.md` lists modules not yet started after a long elapsed time (compare file timestamps if signal is weak)

### 6. Emit the report

Format the output as a single, scannable response. Suggested structure:

```
RDD Status — <project root>
Artifacts: <artifacts_dir>

Setup
─────
target:  ✅ completed (8/8 TDs decided)
map:     ✅ completed (12 modules identified)

Modules                                                                 (legend at bottom)
───────                                                                 ┌─────────────────────────────────┐
                  spec    tests   port              improve   notes     │ ⬜ not started                   │
products          ✅      ✅      ✅                 ✅        wave 2     │ 🔄 in progress (with detail)    │
customers         ✅      ✅      🔄 EP 4/8         ⬜        wave 2     │ ✅ completed                     │
sales             ✅      ⬜      ⬜                 ⬜        wave 4     │ ⏭ skipped / not applicable     │
appointments      🔄 (open questions)  ⬜  ⬜       ⬜        wave 3     └─────────────────────────────────┘
commissions       ⬜      ⬜      ⬜                 ⬜        wave 4
... (remaining modules from MAP)

Currently active
────────────────
• customers — /rdd-port paused at EP-4 of 8. Resume: /rdd-port customers

Blockers
────────
• None

Suggested next step
───────────────────
Resume /rdd-port customers, or start /rdd-spec on a Wave-3 module if you want to parallelize spec/test work ahead.
```

Adjust the layout to fit whatever's actually in the artifacts. The goal is **at-a-glance comprehension**: the user looks at this once and knows where they are.

### 7. Respect missing inputs

If `.rdd.yml` doesn't exist or `artifacts_dir` is empty, don't fail — just say "no RDD work has started yet" and point at `/rdd-target`.

If MAP.md doesn't exist but per-module dirs do (rare — user manually created), report what you find with a note about the missing MAP.

## Anti-patterns to avoid

- **Don't invent state.** If an artifact doesn't exist, don't assume the phase is complete or in progress — report as "not started".
- **Don't write any files.** Status is read-only. No PROGRESS.md, no caches.
- **Don't be silent on partial progress.** A spec with open questions is half-done, not done — say so.
- **Don't suppress blockers.** Failed fix loops, stale work, contradictions — surface them.
- **Don't sort modules by status.** Sort by the order they appear in `MAP.md` (the user's planned wave order). Mixing that order hides the migration shape.
- **Don't bloat the report.** A user runs this skill multiple times a day — keep it dense and scannable.

## Bonus: quick-look mode

If the user says "/rdd-status quick" or "/rdd-status summary", return only the high-level counts:

```
Setup: target ✅  map ✅
Modules: 3/12 done  •  1 in progress  •  8 not started
Currently active: customers (port, EP 4/8)
Blockers: none
```

That's enough for "where am I now" without scanning the full table.
