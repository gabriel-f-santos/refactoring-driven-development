# Refactoring-Driven Development (RDD)

> A Claude Code plugin for **rewriting code with parity guarantees**, from spec through characterization tests to a working port.

RDD is to refactoring/migration what TDD is to greenfield development: a discipline that forces you to **lock current behavior in tests before you touch a line of code**, then rewrite confidently.

It works equally for:

- **Migration between stacks** — Edge Functions → backend service, Express → NestJS, Rails → Phoenix, monolith → services
- **In-place refactor of a legacy module** — same stack, but cleaner code under a behavior-locking test suite
- **Vendor escape** — moving off a managed service to self-hosted with the same observable contract
- **Language port** — JS → TS, Python 2 → Python 3, etc.

## The 4 skills

```
/rdd-map    →  Survey the system, identify modules, propose order
/rdd-spec   →  For one module, capture business rules from code + interview
/rdd-tests  →  Plan characterization tests that lock observable behavior
/rdd-port   →  Implement tests against legacy, then rewrite with parity
```

Each skill **reads the artifacts the previous one wrote**, so you can stop and resume across sessions. Artifacts live under `rdd/` (configurable).

## Workflow

```
                          ┌──────────────┐
                          │  .rdd.yml    │  ← stack config (legacy + target)
                          └──────┬───────┘
                                 │
              ┌──────────────────┼──────────────────┐
              ▼                  ▼                  ▼
       ┌───────────┐      ┌───────────┐      ┌───────────┐
       │ /rdd-map  │      │ /rdd-spec │      │/rdd-tests │
       └─────┬─────┘      └─────┬─────┘      └─────┬─────┘
             ▼                  ▼                  ▼
        rdd/MAP.md      rdd/<mod>/SPEC.md   rdd/<mod>/TESTS.md
                                                    │
                                                    ▼
                                            ┌───────────┐
                                            │ /rdd-port │
                                            └─────┬─────┘
                                                  ▼
                                       new code + green tests
                                       on legacy AND target
```

## Principles

1. **Parity first, refactor later.** First port mimics current behavior exactly. Improvement is a separate step, after green tests prove parity.
2. **Spec before code.** Every module gets a written spec with numbered business rules (BR-01, BR-02...) before any porting.
3. **Characterization, not aspiration.** Tests describe what the system *does*, not what it *should do*. Bug-for-bug parity is the default; intentional behavior changes are tracked explicitly.
4. **Tests that earn their keep.** Every test maps 1:1 to a business rule or observable side effect. No snapshot-of-everything, no "controller calls service", no `expect(x).toBeDefined()`.
5. **Strangler-style cutover.** New code coexists with legacy behind a feature flag. Cutover is gradual and reversible.

## Installation

```bash
claude plugins install github:gabriel-f-santos/refactoring-driven-development
```

Then in any project where you want to use RDD:

```bash
# Copy and edit the config
cp ~/.claude/plugins/refactoring-driven-development/templates/.rdd.yml .rdd.yml
# Open in editor and fill in legacy + target stack
```

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
```

The skills read this file. **Never** hardcode stack assumptions — write them here once.

## Use cases

### Use case 1: Cross-stack migration

You have a legacy backend (Edge Functions, monolithic Rails app, PHP service, etc.) and want to migrate to a new stack module by module.

```bash
/rdd-map                    # produces rdd/MAP.md
/rdd-spec products          # produces rdd/products/SPEC.md
/rdd-tests products         # produces rdd/products/TESTS.md
/rdd-port products          # writes tests + new code, parity verified
```

Repeat per module. Cut over via feature flag when ready.

### Use case 2: In-place refactor

Same stack, but a module accumulated cruft and you want to rewrite it cleanly. The same flow works — `legacy` and `target` in `.rdd.yml` point to the same stack but different source directories (or branches).

### Use case 3: Vendor escape

Moving off a SaaS dependency. Treat the old vendor's API as `legacy`. Treat your replacement as `target`. Run the flow per consumer surface.

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
