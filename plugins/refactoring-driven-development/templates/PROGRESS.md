# {Seq}_{module} — Progress

> Aggregate status for this module. **Single source of truth** for `/rdd-status`.
> Updated by: `/rdd-specify-03` (init), `/rdd-refactor-04` (port), `/rdd-improve-05` (improve).

**Status:** not_started | in_progress | port_complete | improved | done
**Current phase:** spec | tests | harness | lock_legacy | scaffold | port | side_effects | final | improve | done

## Pipeline

| Phase            | Status | Detail                                  |
|------------------|--------|-----------------------------------------|
| spec             | ⬜      | —                                       |
| tests            | ⬜      | —                                       |
| harness          | ⬜      | —                                       |
| lock_legacy      | ⬜      | —                                       |
| scaffold         | ⬜      | —                                       |
| port             | ⬜      | —                                       |
| side_effects     | ⬜      | —                                       |
| final            | ⬜      | —                                       |
| improve          | ⬜      | n/a (lazy — only if /rdd-improve-05 runs) |

Status legend: ⬜ pending • 🔄 in_progress • ✅ completed • ⏭ skipped (.rdd.yml) • ❌ failed

## Tasks (port phase)

| #  | Entry point                | Status | File                            |
|----|----------------------------|--------|---------------------------------|
| 01 | <name>                     | ⬜      | tasks/01_<name>.md              |
| 02 | <name>                     | ⬜      | tasks/02_<name>.md              |
| ...|                            |        |                                 |

> Populated by `/rdd-refactor-04` when the port phase begins. Ordered by the API surface in `spec/SPEC.md`.

## Improve refactors (post-port)

| #  | Refactor                   | Risk   | Status | File                            |
|----|----------------------------|--------|--------|---------------------------------|
| —  | (none yet)                 | —      | —      | —                               |

> Populated by `/rdd-improve-05` if and when it runs. Ordered low-risk → high-risk.

## Upstream gate

| Module               | Required port  | Status        |
|----------------------|----------------|---------------|
| <Seq>_<upstream>     | yes            | ⬜             |

> Captured by `/rdd-refactor-04` Phase 0 from `MAP.md`'s "Depends on" column.
> Module cannot enter port phase until every required upstream is ✅ or ⏭.

## Key paths

- **Spec:** `spec/SPEC.md`
- **Test plan:** `spec/TESTS.md`
- **Per-task progress:** `tasks/NN_<name>.md`
- **Improve plan / refactors:** `improve/IMPROVE.md` + `improve/NN_<name>.md` (lazy)

## History (auto-appended)

> Each transition adds one line. Most recent at top.

- YYYY-MM-DD HH:MM — <skill> — <event> (e.g., "spec phase completed: 30 BRs, 13 errors, 2 OQs")
