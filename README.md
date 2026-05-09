# Refactoring-Driven Development (RDD)

> A Claude Code plugin for **rewriting code with parity guarantees**, from spec through characterization tests to a working port.

RDD is to refactoring/migration what TDD is to greenfield development: a discipline that forces you to **lock current behavior in tests before you touch a line of code**, then rewrite confidently.

> **A note on "TDD".** Classical TDD (Beck) writes a failing test for code that doesn't exist yet, then makes it pass. RDD applies the same test-first principle to **legacy code that already exists** — what Michael Feathers called *characterization testing*. Tests describe what the legacy *does*, not what it *should do*. The discipline is the same (red → green → refactor); the starting point differs.

It works equally for:

- **Migration between stacks** — Edge Functions → backend service, Express → NestJS, Rails → Phoenix, monolith → services
- **In-place refactor of a legacy module** — same stack, but cleaner code under a behavior-locking test suite
- **Vendor escape** — moving off a managed service to self-hosted with the same observable contract
- **Language port** — JS → TS, Python 2 → Python 3, etc.
- **Idiomatic improvement** — clean up a parity-correct module after porting, with tests as a safety net

## The 6 skills

**Core pipeline (4 skills):**

```
/rdd-specify-01        →  Decide where to migrate: stack, architecture, conventions
/rdd-map-codebase-02   →  Survey the legacy, identify modules, propose order
/rdd-specify-03        →  For one module, capture business rules from code + interview
/rdd-refactor-04       →  Plan characterization tests, lock legacy, port with parity
```

**Optional (2 skills):**

```
/rdd-improve-05        →  After parity, refactor the new code idiomatically — tests guard parity
/rdd-status            →  Show migration progress across all modules and phases
```

Three skills to **specify and map** (specify-01 → map-codebase-02 → specify-03) before one skill to **refactor with TDD** (refactor-04, which merges test planning + parity port), with one to **polish** (improve-05) and one to **observe** (status). Each reads the artifacts the previous one wrote, so you can stop and resume across sessions. Artifacts live under `rdd/` (configurable).

## Workflow

```
        ┌──────────────────┐
        │ /rdd-specify-01  │  ← decide architecture, framework, conventions
        └────────┬─────────┘
                 ▼
           rdd/TARGET.md  +  populates .rdd.yml target block
                 │
                 ▼
        ┌────────────────────────┐
        │ /rdd-map-codebase-02   │  ← survey legacy with target in mind
        └────────┬───────────────┘
                 ▼
            rdd/MAP.md
                 │
                 ▼  (per module, repeat)
        ┌──────────────────┐      ┌──────────────────┐      ┌──────────────────┐
        │ /rdd-specify-03  │  →   │ /rdd-refactor-04 │  →   │ /rdd-improve-05  │
        └────────┬─────────┘      └────────┬─────────┘      └────────┬─────────┘
                 ▼                         ▼                         ▼
          rdd/<m>/SPEC.md      rdd/<m>/TESTS.md (Phase 1)    idiomatic code
                               + parity-correct code         (same green tests
                               + green tests on legacy        still pass)
                                 AND target

           /rdd-status  ← run anytime to see where each module is
```

## Principles

1. **Parity first, refactor later.** First port mimics current behavior exactly. Improvement is a separate step, after green tests prove parity.
2. **Decide before mapping.** Target architecture (TD-01, TD-02...) is decided up front and recorded with rationale. Module grouping and test posture flow from those decisions.
3. **Spec before code.** Every module gets a written spec with numbered business rules (BR-01, BR-02...) before any porting.
4. **Characterization, not aspiration.** Tests describe what the system *does*, not what it *should do*. Bug-for-bug parity is the default; intentional behavior changes are tracked explicitly.
5. **Tests that earn their keep.** Every test maps 1:1 to a business rule or observable side effect, written using AC template formulas (`[METHOD] [/path] with [input] returns [status] with [body]`). No snapshot-of-everything, no "controller calls service", no `expect(x).toBeDefined()`.
6. **Validate before generating.** Each skill runs a pre-flight check against its inputs (config consistency, cross-document contradictions, missing decisions) before writing a single line of output.
7. **Resumable execution.** Long-running ports persist state in a progress file (`REFACTOR.progress.md`) and a per-entry-point task list. Stop after each entry point and wait for explicit "Continuar?" — unless the user opted into continuous mode.
8. **Fix-loop discipline.** When tests fail during porting, max 3 focused attempts before escalating. No weakening tests, no skipping, no swallowing errors.
9. **Strangler-style cutover.** New code coexists with legacy behind a feature flag. Cutover is gradual and reversible.

## Installation

Inside Claude Code, add the marketplace and install the plugin:

```
/plugin marketplace add gabriel-f-santos/refactoring-driven-development
/plugin install refactoring-driven-development@gabriel-f-santos
```

Verify the skills are available by typing `/` in Claude Code — you should see `/rdd-specify-01`, `/rdd-map-codebase-02`, `/rdd-specify-03`, `/rdd-refactor-04`, `/rdd-improve-05`, and `/rdd-status`.

Then in any project where you want to use RDD, just invoke `/rdd-specify-01` — the skill auto-creates `.rdd.yml` from the bundled template if it doesn't exist and walks you through filling in legacy + target stack.

## Configuration: `.rdd.yml`

```yaml
legacy:
  stack: "Supabase Edge Functions (Deno)"
  source: "supabase/functions/"
  database: "Postgres (Supabase)"
  notes: "RLS policies in migration files; some shared utilities in _shared/"

target:
  stack: "NestJS + Fastify"
  source: "apps/api/src/"
  test_framework: "Vitest"
  test_strategy: "integration-first; testcontainers Postgres; mocks only at HTTP boundary"

artifacts_dir: "rdd/"

conventions:
  business_rule_prefix: "BR"
  module_dir_pattern: "rdd/{module}/"

# Optional: skip /rdd-specify-01 when target stack is already established
# (e.g., in-place refactor of a consolidated codebase). The skill produces
# a minimal TARGET.md focused on conventions.
skip_target: false
```

The skills read this file. **Never** hardcode stack assumptions — write them here once.

## Lightweight mode

The full pipeline is calibrated for **high-stakes work** — production migrations, multi-month rewrites, multi-tenant SaaS. For smaller scopes, skip what doesn't pay for itself:

| Scope | Skip | Why |
|-------|------|-----|
| In-place refactor with target = legacy stack | Set `skip_target: true` | No architectural decisions to make; conventions already established |
| Single small module (≤5 entry points) | Skip `/rdd-map-codebase-02` | Module boundary is obvious; just go straight to `/rdd-specify-03` |
| Pure cosmetic refactor (rename, extract method) inside an already-tested module | Skip everything; use tests directly | RDD overhead doesn't pay off for a 10-minute change |
| Greenfield code | Don't use RDD | RDD assumes legacy code to characterize; for new code use spec-kit or similar |

**Heuristic:** if the change touches >300 lines of legacy code OR has >2 reasonable architectures OR will be in production for >12 months, run the full pipeline. Otherwise, drop phases that don't earn their keep.

## Use cases

### Use case 1: Cross-stack migration

You have a legacy backend (Edge Functions, monolithic Rails app, PHP service, etc.) and want to migrate to a new stack module by module.

```bash
/rdd-specify-01                 # decide target stack, architecture, conventions → rdd/TARGET.md
/rdd-map-codebase-02                    # survey the legacy with target in mind → rdd/MAP.md
/rdd-specify-03 products          # → rdd/products/SPEC.md
/rdd-refactor-04 products         # → rdd/products/TESTS.md
/rdd-refactor-04 products          # writes tests + new code, parity verified
```

Run `/rdd-specify-01` and `/rdd-map-codebase-02` once. Repeat the spec → tests → port loop per module. Cut over via feature flag when ready.

### Use case 2: In-place refactor

Same stack, but a module accumulated cruft and you want to rewrite it cleanly. The same flow works — `legacy` and `target` in `.rdd.yml` point to the same stack but different source directories (or branches).

### Use case 3: Vendor escape

Moving off a SaaS dependency. Treat the old vendor's API as `legacy`. Treat your replacement as `target`. Run the flow per consumer surface.

### Use case 4: Idiomatic improvement after porting

You ran `/rdd-refactor-04 products` and the new module is parity-correct but ugly — direct copy of legacy structure, repeated code, no value objects. Run `/rdd-improve-05 products` to clean up incrementally. The same characterization tests from `/rdd-refactor-04` guard parity: any refactor that breaks observable behavior shows up immediately.

```bash
/rdd-improve-05 products       # → rdd/products/IMPROVE.md (refactor plan), then incremental refactors
```

### Use case 5: Tracking progress across many modules

For a multi-module migration spanning weeks or months, you need a quick way to see where each module is. Run `/rdd-status` anytime:

```bash
/rdd-status                 # reads existing artifacts, prints a per-module phase table
```

No persistent state file — `/rdd-status` infers progress from what's on disk.

## What this is not

- **Not a code generator from scratch.** RDD assumes you have working legacy code to characterize. For greenfield work, use spec-kit or similar.
- **Not a linter or autofix.** It coordinates a human + LLM workflow; it doesn't blindly transform code.
- **Not microservices-specific.** The Strangler Fig pattern that inspired part of this is *one* cutover strategy among many.

## Anti-pattern: tests that don't earn their keep

RDD has strong opinions on what tests to write — and not write — during refactoring. Summary:

| Write                                | Don't write                              |
|--------------------------------------|------------------------------------------|
| End-to-end use cases from spec       | "Controller calls service"               |
| Domain invariants                    | DTO validation already enforced by lib   |
| Observable error paths (403, 422)    | Snapshots of mutable JSON                |
| Idempotency of webhooks              | `expect(x).toBeDefined()` without intent |
| Property-based for calculations      | Mock-of-mock-of-mock                     |
| Boundary cases discovered in code    | Tests that break on every refactor       |

If a test breaks during refactor *without changing observable behavior*, it was testing implementation. Delete it.

## Contributing

PRs welcome. Especially:

- Examples of RDD applied to other migration scenarios (add to `examples/`)
- Improvements to skill prompts based on real-world usage
- Translations of the skill prompts (current: English; the methodology is language-agnostic)

## License

MIT
