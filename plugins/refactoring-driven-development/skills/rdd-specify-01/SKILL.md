---
name: rdd-specify-01
description: Use this skill at the very start of a refactoring or migration project to make the forward-looking architectural decisions (stack, framework, conventions) before any legacy survey or refactor begins. First step in the RDD pipeline. Triggers on "where should I migrate to", "decide target stack", "choose framework", "design the target architecture". Produces rdd/TARGET.md with numbered technical decisions (TD-NN) and populates .rdd.yml.
---

# rdd-specify-01 — Decide where you're going

You are helping the user make the **forward-looking architectural decisions** for a refactoring or migration project. This skill runs **before** `/rdd-map-codebase-02` because chosen target shapes how legacy modules are grouped, which test framework is used, and which cutover strategies apply.

This is the **only RDD skill where decisions about the target are made**. After this, decisions are locked into `TARGET.md` and `.rdd.yml`; downstream skills consume them as constraints.

## Procedure

### 1. Initialize configuration and check skip_target

Check for `.rdd.yml` in the project root. If absent, copy from `templates/.rdd.yml`. If `legacy.*` is unfilled, ask for it now (just `legacy.stack` and `legacy.source` — enough for the survey step).

**Check `skip_target`.** If `.rdd.yml` has `skip_target: true`, switch to **minimal mode** (skip steps 2–8 and produce a conventions-only TARGET.md):

- Confirm with the user: *"`skip_target: true` detected. Producing minimal TARGET.md focused on conventions only — no architecture decisions or candidate comparison. Continue?"*
- If confirmed, jump to step 9 (Document) but write only the **Why migrating** paragraph and the **Conventions** section — skip Technical Decisions, Cross-cutting concerns, and Decisions Summary
- Then jump to step 11 (Auto-populate `.rdd.yml`) using values already present in the file (or asking the user for any gaps)

This is for in-place refactors of consolidated stacks: no architectural choices to make, just record the conventions the rewrite will follow.

### 2. Run a focused constraints interview

**Tight: 5–8 questions, not open-ended brainstorming.** Cover:

- **Why are you migrating?** (cost, vendor lock-in, performance, scale, DX, hiring, compliance) — most important question; the rest follow
- **Team profile.** Solo, small (2–5), or larger? What languages/frameworks do they already know well?
- **Deployment constraints.** Cloud preference? On-prem? Data residency / compliance? Existing infra to integrate with?
- **Budget posture.** Penny-pinching (free tiers + VPS), comfortable (managed), enterprise?
- **Time horizon.** Ship in 3 months? Willing to invest 1–2 years?
- **Non-negotiables.** Stack components that **must** be kept (existing DB, IdP, language)
- **Existing pain points** in legacy that target must fix (cold starts, cost per request, slow tests, deploy friction)

If the user already knows the target, skip ahead to step 5 — but still produce TARGET.md (decision capture, even if quick).

### 3. Survey the legacy at high level

Not the deep inventory `/rdd-map-codebase-02` performs — **15–30 minutes of reading max**. Just enough to:

- Identify legacy architectural style (monolith, function-per-endpoint, services, event-driven)
- Spot integration points that constrain target choice (queues, third-party APIs, webhooks, scheduled jobs)
- Estimate domain complexity (CRUD-heavy vs business-rule-heavy vs streaming/stateful) — favors different stacks
- Note language(s) used and binary/FFI dependencies that affect portability

### 4. Identify technical decisions (TD-NN)

Read each architectural concern and ask: *"Is there more than one reasonable way to handle this given the user's constraints?"* If yes, it's a technical decision.

Decisions appear in recurring categories:

- **Backend stack** — language, framework, runtime
- **Architecture pattern** — monolithic, modular monolith, microservices, serverless
- **Persistence** — keep current DB, migrate, dual-write
- **ORM / data access** — driver, ORM, query builder
- **Authentication** — keep current IdP, replace, hybrid
- **Hosting / deployment** — VPS, PaaS, K8s, serverless platform
- **Test strategy** — framework + integration-vs-unit posture
- **Cutover strategy** — strangler with flags, big-bang, dual-write
- **Async / queue** — broker choice, library
- **Observability** — log aggregator, tracing
- **Validation library** — when there are real alternatives

**Skip** decisions already made in `.rdd.yml`, decisions with one obviously dominant option given constraints, and pure conventions (kebab-case file names, etc.) — those go in the Conventions section, not as TDs.

Number the decisions: **TD-01, TD-02, ...**. Note dependencies (e.g., "TD-03 depends on TD-02").

### 5. Dispatch Context7 lookups in parallel

For each TD that involves specific libraries, frameworks, or services, fire `mcp__context7__resolve-library-id` followed by `mcp__context7__query-docs` for each candidate. **Issue all calls in a single message.** Do not look libraries up sequentially as you write each TD — batch them upfront and refer to results.

This grounds recommendations in current documentation, not stale training data. Versions, APIs, and library status change.

If a TD involves comparing services (Railway vs Fly vs Render), use web search instead of Context7.

### 6. Generate 2–4 options per TD

**Rules:**
- **<2 options is not a decision** — don't write a TD with one option; either drop it or expand
- **>4 options is noise** — pre-filter to the realistic candidates given constraints
- **Don't include obviously-inadequate options** to pad the list

For each option, document:

- **Name** — concise label (`NestJS + Fastify adapter`, not `Node.js framework`)
- **How it works** — 2–3 sentences max
- **Pros** — concrete, tied to this user's constraints (not generic praise)
- **Cons** — concrete, honest about cost, risk, complexity

### 7. Write recommendations honestly

A recommendation is a **suggestion**, not a decision. It must:

- Be justified by stated constraints from step 2, not by generic preference
- Consider what's already decided in earlier TDs of this project
- Be explicit about what's gained and what's lost
- Admit when two options are equivalent given the constraints

**BAD recommendations:**

- ❌ "JWT is more modern."
- ❌ "Everyone uses NestJS for this."
- ❌ "Drizzle is faster." *(faster than what? does it matter at projected scale?)*
- ❌ "Recommend Postgres because it's reliable." *(no constraint cited)*

**GOOD recommendations:**

- ✅ "JWT + refresh in DB enables rotation per RFC 9700 without adding Redis as an auth dependency, since PostgreSQL is already in the stack."
- ✅ "NestJS + Fastify adapter — module structure fits the 88-domain ERP scope; DI cuts test fixture boilerplate; Fastify adapter holds the ~28k req/s perf the user listed as a constraint."
- ✅ "Drizzle over Prisma — Better Auth (chosen in TD-04) ships an official Drizzle adapter; Prisma adapter is community-maintained. Constraint: solo dev wants minimal third-party glue."
- ✅ "Equivalent given your constraints — both Vitest and Jest meet the integration-test posture you stated. Recommend Vitest only because target uses Vite-style tooling already."

If recommendation is genuinely "either is fine", say so. Forcing a recommendation on a tie wastes the user's decision-making bandwidth.

### 8. Discuss tradeoffs and let the user decide

Present TDs as a comparison. Surface tradeoffs they may not have considered. **Do not decide for the user** — they own consequences for years.

If pushed to "just pick", make a recommendation **with explicit rationale tied to their constraints**, but flag it as your recommendation, not an objective best.

### 9. Document in `TARGET.md`

Create `{artifacts_dir}/TARGET.md` using `templates/TARGET.md`. Structure:

- **Why migrating** — one paragraph from step 2
- **Technical Decisions** — all TDs numbered, each with Context, Options, Recommendation, Decision field (left blank for user)
- **Conventions** — naming, folder structure, error handling style, validation, logging, multi-tenancy boundary, money/time handling (these are not TDs because they don't have meaningful alternatives)
- **Cross-cutting concerns** — observability, audit, rate limiting, idempotency, jobs, secrets
- **Out of scope** — architectural decisions explicitly deferred
- **Open questions** — anything that blocks downstream skills
- **Decisions Summary** — table at the end mapping TD-NN → Choice (auditable single-page view)

Use **incremental drafting**: write the file in pieces (header → why → first TD → second TD → ... → conventions → summary). Don't try to one-shot a long file — append section by section.

If a decision is missing context and was not caught in step 4, insert an inline `[DECIDE: option A | option B — brief context]` marker and continue. At the end of drafting, surface all markers to the user via `AskUserQuestion` (up to 4 per call) and apply answers with targeted `Edit`.

### 10. User fills the Decision fields

Walk through each TD with the user. Their answer goes in the `**Decision:** _<choice>_` line of each TD and the Decisions Summary row.

You should never write the Decision field on your own. The user owns it.

### 11. Auto-populate `.rdd.yml`

Update the `target` block of `.rdd.yml` from `TARGET.md`:

- `target.stack` — short string from the chosen stack TD
- `target.source` — proposed source dir (ask user to confirm; default like `apps/api/src/`)
- `target.test_framework` — exact framework from the test TD
- `target.test_strategy` — short summary of the test posture from Conventions

Do **not** invent values — if `TARGET.md` doesn't have it, ask the user.

### 12. Hand-off

Tell the user:

- Path to `TARGET.md` and that all Decision fields are filled
- Confirmation that `.rdd.yml` is filled
- Next step: `/rdd-map-codebase-02` will use the target choice to inform module grouping and migration order

## In-place refactor mode

If target = legacy stack (clean rewrite without changing tech), this skill still runs but is shorter:

- Skip the candidate-generation step (no stack alternatives to compare)
- Most TDs collapse — skip them
- Focus on **conventions** that the rewrite will follow (naming, folder structure, error handling, validation library if changing)
- `TARGET.md` is mostly the Conventions section + a short "Why rewriting" paragraph

## When this skill is needed

**Always**, at the start of any RDD project. There are no shortcuts — even the in-place case produces a TARGET.md (shorter, but still a record).

The only legitimate skip is when the user already has an equivalent decision document from another methodology. In that case, copy its decisions into TARGET.md format so downstream skills can read them uniformly.

## Anti-patterns to avoid

- **Don't pick the trendy default.** "Everyone uses X" is not a justification.
- **Don't decide architecture without input.** This is the user's call. You surface tradeoffs.
- **Don't underestimate boring choices.** Postgres + a monolithic framework solves more problems than people admit.
- **Don't conflate stack with pattern.** "NestJS" is a framework; "modular monolith" is a pattern. Decide them separately (separate TDs).
- **Don't decide things downstream skills should decide.** Module boundaries → `/rdd-map-codebase-02`. Per-module API → `/rdd-specify-03`. Specific tests → `/rdd-refactor-04`. This skill stays at the architecture level.
- **Don't write `TARGET.md` without rationale.** A doc that says "we use NestJS" without "because [stated constraint]" is useless six months later when someone asks why.
- **Don't fill in Decision fields yourself.** They're left blank for the user.
- **Don't pad to 4 options.** 2 honest options beats 4 with one strawman.

## When to spawn sub-agents

If the legacy stack is one you're less familiar with and the comparison requires up-to-date framework info, use the web-researcher agent to gather current data on candidate frameworks. Pass it the user's constraints and ask for honest comparison — strengths, weaknesses, recent breaking changes, community trajectory. For specific library APIs, prefer Context7 over web search.
