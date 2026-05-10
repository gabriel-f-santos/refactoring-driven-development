---
name: rdd-map-codebase-02
description: Use this skill after rdd-specify-01 to survey the legacy codebase, identify module boundaries, and propose a safe migration order. Second step in the RDD pipeline. Triggers on "map the legacy", "map the codebase", "where do I start", "what modules do I have", "plan the migration". Produces rdd/MAP.md.
---

# rdd-map-codebase-02 — Survey the legacy system

You are mapping a legacy codebase as the **second step** of a Refactoring-Driven Development workflow (after `/rdd-specify-01`). Your job is to identify migration boundaries (modules), their dependencies, and propose a safe order shaped by the chosen target architecture. You are not writing implementation code in this step.

## Procedure

### 1. Load configuration and target decision

Read in order:

- `.rdd.yml` from the project root — for `legacy.*`, `target.*`, and `artifacts_dir`.
- `{artifacts_dir}/TARGET.md` — the target architecture decision from `/rdd-specify-01`.

If either is missing, instruct the user to run `/rdd-specify-01` first. **Do not proceed without `TARGET.md`** — module grouping and migration order depend on the target architecture pattern (a modular monolith and a microservices target produce different maps for the same legacy).

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

- Every TD in the **Decisions Summary** has a `Choice` filled (not `<pending>`). If any TD is unresolved, stop and ask the user to complete `/rdd-specify-01` first — module grouping depends on the architecture pattern decision.
- The **chosen architecture pattern** (TD covering monolith vs microservices vs serverless) is explicit. The MAP grouping rules in step 6 depend on this.
- The **chosen cutover strategy** is explicit. Migration ordering and risk classification depend on it.

**Cross-document contradictions:**

- `legacy.stack` describes a stack the chosen target architecture cannot meaningfully replace (e.g., if legacy is a desktop app and target is a web backend, the migration scope is mis-defined — escalate before mapping)
- Decisions in `TARGET.md` reference modules or capabilities not present in `legacy.source` (rare, but worth flagging)
- The `target.test_framework` is set in `.rdd.yml` (downstream skills depend on it; if missing, flag)

**If any issue is found:** stop. Quote the conflicting statements or point to the missing field. Wait for the user to resolve (typically by re-running `/rdd-specify-01` or editing `.rdd.yml`). Re-validate before proceeding.

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

#### 6.1 — Assign sequence numbers (`Seq`)

Every module gets a 3-digit zero-padded sequence number (`Seq`) that defines a **total order** for the migration. Downstream skills (`/rdd-specify-03`, `/rdd-refactor-04`, `/rdd-improve-05`, `/rdd-status`) use `Seq` to:

- Locate the module's artifacts directory (folder name is `NNN_<module>/`)
- Sort modules and pick the next eligible one
- Enforce upstream-port-completed gates (a module with `Seq` = `010` cannot port until every `Seq < 010` is completed or explicitly skipped)

**Numbering rule:**

- Use **sequential numbering**: `001`, `002`, `003`, ..., `099`. Zero-padded to 3 digits. Same convention as Django migrations and similar ordered-list tools — simpler than gap-based numbering, and inserts mid-flight are rare in legacy migrations (the legacy code is fixed; if a missed module surfaces later, re-run `/rdd-map-codebase-02` to renumber, or `git mv` the affected modules in one commit).
- **Order by dependency (topological), not by wave or risk.** A module's `Seq` must be greater than every `Seq` it depends on. `auth` typically gets a low `Seq` because many modules depend on it; `reports` and `admin` typically get high `Seq` because they read from many modules.
- **Tiebreak alphabetically** when two modules at the same dependency level could go in either order.
- **Wave is separate from Seq, and only relevant if TD-08=B.** The wave column captures **cutover risk** ("read-only first", "auth last for blast radius") — a deployment concern relevant ONLY when the user chose `TD-08 = B` (strangler-fig with feature flags) in `TARGET.md`. With `TD-08 = A` (big-bang side-by-side, the default), cutover is one event at the end and there is no per-module rollout order to plan — `Seq` alone defines port order. Don't conflate Seq and Wave even when both apply; mixing them produces cross-wave dependencies that need `.rdd.yml skipped:` workarounds.
- The same `Seq` must not be assigned to two modules.

**Do not create directories in this step.** `Seq` is recorded in `MAP.md` only. Module directories (`{artifacts_dir}/NNN_<module>/`) are created lazily by `/rdd-specify-03` when each module is specced. This keeps the filesystem clean of empty placeholders and lets the user adjust `Seq` in `MAP.md` text before any spec work commits to disk.

### 7. Write `MAP.md`

Create `{artifacts_dir}/MAP.md` using the template in `templates/MAP.md`. It must contain:

- **Summary** — total entry points, modules identified, estimated calendar effort
- **Module table** — `Seq`, name, entry-point count, domain summary, complexity (S/M/L/XL), risk (low/med/high), dependencies. **`Seq` is the leftmost column** so the migration order is visually obvious.
- **Dependency notes** — any cycles, hubs, or surprises
- **Wave plan (cutover risk)** — **conditional**. Only include this section if `TD-08 = B` (strangler-fig with feature flags) in `TARGET.md`. With `TD-08 = A` (big-bang side-by-side, default), cutover is one event after final verification — there is no per-wave rollout to plan. When the section IS included, it tabulates which modules' flags flip together based on rollback risk, blast radius, etc.
- **Open questions** — things you couldn't determine from code alone (ask the user)

### 8. Hand-off

Tell the user:

- Brief summary of what you found (counts, surprises, blockers)
- Path to `MAP.md`
- The lowest-`Seq` module — that's where `/rdd-specify-03 <module>` (or batch) should start
- Reminder: module directories will be created on demand by `/rdd-specify-03`; `MAP.md`'s `Seq` column is the source of truth for ordering until then

## Anti-patterns to avoid

- **Don't read every file in detail.** This is a survey. Identify boundaries; deep-read happens in `/rdd-specify-03`.
- **Don't propose specific implementation choices** for the new system. That's not your job here.
- **Don't merge clearly distinct domains** just because they share files. Flag the entanglement; let the user decide.
- **Don't skip the configuration check.** Without `.rdd.yml`, downstream skills break.
- **Don't create module directories.** Folder layout is filesystem-true only after `/rdd-specify-03` runs. This skill writes `MAP.md` and stops — directories are a downstream concern.
- **Don't assign duplicate `Seq` values.** Each module is unique. If two modules end up at the same `Seq`, you forgot to apply the within-wave topological/alphabetical tiebreak.
- **Don't conflate `Seq` with wave/cutover risk.** `Seq` is dependency order (topological). Wave is rollout strategy. Mixing them produces cross-wave dependencies that need `.rdd.yml skipped:` workarounds for ordering — those should only be needed for genuinely-skipped modules (deprecated, out of scope), not for ordering tension.

## When to spawn sub-agents

If the legacy codebase has more than ~50 entry points or ~500 files, use the Explore agent for the inventory step with a very-thorough prompt. Pass it the legacy source directory and ask it to return a structured list of entry points + their files + their external integrations. Then synthesize the module grouping yourself from its result.
