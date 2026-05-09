---
name: rdd-map
description: Use this skill when starting a refactoring or migration project to survey the legacy system, identify module boundaries, and propose a safe migration order. Triggers on "map the legacy", "where do I start", "what modules do I have", "plan the migration". Produces rdd/MAP.md.
---

# rdd-map — Survey the legacy system

You are mapping a legacy codebase as the **first step** of a Refactoring-Driven Development workflow. Your job is to identify migration boundaries (modules), their dependencies, and propose a safe order. You are not writing implementation code in this step.

## Procedure

### 1. Load configuration and target decision

Read in order:

- `.rdd.yml` from the project root — for `legacy.*`, `target.*`, and `artifacts_dir`.
- `{artifacts_dir}/TARGET.md` — the target architecture decision from `/rdd-target`.

If either is missing, instruct the user to run `/rdd-target` first. **Do not proceed without `TARGET.md`** — module grouping and migration order depend on the target architecture pattern (a modular monolith and a microservices target produce different maps for the same legacy).

Key fields you'll use:
- `legacy.source` — where to scan
- `legacy.stack` — informs file-reading patterns and entry-point heuristics
- `TARGET.md` — read the **architecture pattern**, **cutover strategy**, and **module structure conventions**; these shape the MAP
- `artifacts_dir` — where to write `MAP.md`

### 2. Validate inputs (pre-flight)

Before scanning a single file, run these checks against the inputs you just loaded. Catch problems that would otherwise produce a confused MAP. Stop and surface findings to the user instead of guessing.

**Configuration consistency:**

- `legacy.source` path exists in the project
- `legacy.stack` description is non-empty and plausible (matches what's actually in `legacy.source` — e.g., if `legacy.stack` says "Rails" but the directory has only TS files, flag it)
- `artifacts_dir` is writable

**`TARGET.md` completeness:**

- Every TD in the **Decisions Summary** has a `Choice` filled (not `<pending>`). If any TD is unresolved, stop and ask the user to complete `/rdd-target` first — module grouping depends on the architecture pattern decision.
- The **chosen architecture pattern** (TD covering monolith vs microservices vs serverless) is explicit. The MAP grouping rules in step 6 depend on this.
- The **chosen cutover strategy** is explicit. Migration ordering and risk classification depend on it.

**Cross-document contradictions:**

- `legacy.stack` describes a stack the chosen target architecture cannot meaningfully replace (e.g., if legacy is a desktop app and target is a web backend, the migration scope is mis-defined — escalate before mapping)
- Decisions in `TARGET.md` reference modules or capabilities not present in `legacy.source` (rare, but worth flagging)
- The `target.test_framework` is set in `.rdd.yml` (downstream skills depend on it; if missing, flag)

**If any issue is found:** stop. Quote the conflicting statements or point to the missing field. Wait for the user to resolve (typically by re-running `/rdd-target` or editing `.rdd.yml`). Re-validate before proceeding.

**If no issues:** proceed.

### 3. Inventory the legacy code

Use codebase exploration tools (Glob, Grep, Read, or the Explore agent for large codebases):

- List every file under `legacy.source`.
- Identify cross-cutting utilities (shared helpers, auth middleware, logging — usually in `_shared/`, `lib/`, `utils/`, `common/`).
- For backend code: identify entry points (HTTP handlers, queue consumers, scheduled jobs, webhook receivers).
- For each entry point, capture: name, purpose (1 line), files involved, external integrations called, database tables touched.

If the codebase is large (>50 entry points or >500 files), spawn the Explore agent with a very-thorough prompt to do this inventory.

### 4. Group entry points into modules

Heuristics to group:

- **Folder structure** — usually the strongest signal.
- **Naming prefixes** — `agent-products`, `agent-products-wa`, `list-products` likely belong together.
- **Shared database tables** — operations that read/write the same tables are usually one module.
- **Shared business vocabulary** — "sales", "vendas", "orders" likely cluster.

A module is the **unit of migration** — the smallest chunk you can cut over independently. Aim for modules with:

- 5–20 entry points each (rough)
- A coherent domain noun (`products`, `sales`, `commissions`)
- Minimal dependencies on other not-yet-migrated modules

If you find one giant blob (e.g., one shared "utils" module), call it out explicitly — it will block migration and may need pre-emptive splitting.

### 5. Detect dependencies

For each module, find which other modules it depends on:

- Calls into another module's entry points
- Reads tables owned by another module
- Imports utilities from another module

Build a dependency relationship list. Look for:

- **Cycles** — flag them; they make ordering hard
- **Hubs** — modules many others depend on; migrate late or migrate carefully

### 6. Propose a migration order

**Adjust grouping to the target architecture pattern from `TARGET.md`:**

- **Modular monolith target** — group by domain (one module per bounded context). This is the most common case.
- **Microservices target** — same domain grouping, but each module also gets a service boundary call-out (data ownership, API surface).
- **Serverless target** — finer granularity; one entry point may be one function. Group by deployment unit, not just by domain.
- **In-place refactor** — group by current code structure since the move is in-stack.

Default ordering by **risk × dependency** (lower risk first):

1. **Read-only / reporting** — analytics, dashboards, list endpoints with no writes
2. **Simple CRUD** — well-isolated entities, few external calls
3. **CRUD with side effects** — writes that trigger notifications, calendar sync, etc.
4. **Transactional core** — sales, orders, payments — multi-table writes, business rules
5. **External integrations** — webhook receivers, third-party APIs (Stripe, payment gateways, messaging)
6. **AI / streaming / stateful** — chat assistants, agents, streaming endpoints
7. **Admin / internal tools** — low-traffic, can wait
8. **Auth** — last (or near-last). Highest blast radius if it breaks.

Adjust based on dependencies (a module everyone depends on must be available; if it's high-risk, mitigate).

### 7. Write `MAP.md`

Create `{artifacts_dir}/MAP.md` using the template in `templates/MAP.md`. It must contain:

- **Summary** — total entry points, modules identified, estimated calendar effort
- **Module table** — name, entry-point count, domain summary, complexity (S/M/L/XL), risk (low/med/high), dependencies
- **Dependency notes** — any cycles, hubs, or surprises
- **Migration order** — numbered list of waves with rationale
- **Open questions** — things you couldn't determine from code alone (ask the user)

### 8. Hand-off

Tell the user:

- Brief summary of what you found (counts, surprises, blockers)
- Path to `MAP.md`
- Suggested first module to spec (`/rdd-spec <module>`)

## Anti-patterns to avoid

- **Don't read every file in detail.** This is a survey. Identify boundaries; deep-read happens in `/rdd-spec`.
- **Don't propose specific implementation choices** for the new system. That's not your job here.
- **Don't merge clearly distinct domains** just because they share files. Flag the entanglement; let the user decide.
- **Don't skip the configuration check.** Without `.rdd.yml`, downstream skills break.

## When to spawn sub-agents

If the legacy codebase has more than ~50 entry points or ~500 files, use the Explore agent for the inventory step with a very-thorough prompt. Pass it the legacy source directory and ask it to return a structured list of entry points + their files + their external integrations. Then synthesize the module grouping yourself from its result.
