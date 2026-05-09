---
name: rdd-map
description: Use this skill when starting a refactoring or migration project to survey the legacy system, identify module boundaries, and propose a safe migration order. Triggers on "map the legacy", "where do I start", "what modules do I have", "plan the migration". Produces rdd/MAP.md.
---

# rdd-map — Survey the legacy system

You are mapping a legacy codebase as the **first step** of a Refactoring-Driven Development workflow. Your job is to identify migration boundaries (modules), their dependencies, and propose a safe order. You are not writing implementation code in this step.

## Procedure

### 1. Load configuration

Read `.rdd.yml` from the project root. If it does not exist:

- Look for `templates/.rdd.yml` in the plugin directory (typically `~/.claude/plugins/refactoring-driven-development/templates/`).
- Copy it to the project root.
- Open the file and ask the user to fill in the `legacy` and `target` blocks. **Do not proceed until the file is filled.**

The configuration tells you:
- `legacy.source` — where the legacy code lives
- `legacy.stack` — what tech it uses (informs your file-reading patterns)
- `target.source` — where new code will go (used by later skills, but record now)
- `artifacts_dir` — where to write `MAP.md` and per-module artifacts (default `rdd/`)

### 2. Inventory the legacy code

Use codebase exploration tools (Glob, Grep, Read, or the Explore agent for large codebases):

- List every file under `legacy.source`.
- Identify cross-cutting utilities (shared helpers, auth middleware, logging — usually in `_shared/`, `lib/`, `utils/`, `common/`).
- For backend code: identify entry points (HTTP handlers, queue consumers, scheduled jobs, webhook receivers).
- For each entry point, capture: name, purpose (1 line), files involved, external integrations called, database tables touched.

If the codebase is large (>50 entry points or >500 files), spawn the Explore agent with a very-thorough prompt to do this inventory.

### 3. Group entry points into modules

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

### 4. Detect dependencies

For each module, find which other modules it depends on:

- Calls into another module's entry points
- Reads tables owned by another module
- Imports utilities from another module

Build a dependency relationship list. Look for:

- **Cycles** — flag them; they make ordering hard
- **Hubs** — modules many others depend on; migrate late or migrate carefully

### 5. Propose a migration order

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

### 6. Write `MAP.md`

Create `{artifacts_dir}/MAP.md` using the template in `templates/MAP.md`. It must contain:

- **Summary** — total entry points, modules identified, estimated calendar effort
- **Module table** — name, entry-point count, domain summary, complexity (S/M/L/XL), risk (low/med/high), dependencies
- **Dependency notes** — any cycles, hubs, or surprises
- **Migration order** — numbered list of waves with rationale
- **Open questions** — things you couldn't determine from code alone (ask the user)

### 7. Hand-off

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
