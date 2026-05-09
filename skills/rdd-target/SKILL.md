---
name: rdd-target
description: Use this skill at the very start of a refactoring or migration project to decide the target architecture, framework, and conventions before any legacy survey or porting begins. Triggers on "where should I migrate to", "decide target stack", "choose framework", "design the target architecture". Produces rdd/TARGET.md and populates .rdd.yml.
---

# rdd-target — Decide where you're going

You are helping the user make the **forward-looking architectural decision** for a refactoring or migration project. This skill runs **before** `/rdd-map` because the chosen target informs how legacy modules are grouped, what test framework is used, and which cutover strategies apply.

This is the **only RDD skill where decisions about the target are made**. After this, decisions are locked into `TARGET.md` and `.rdd.yml`; downstream skills consume them as constraints.

## Procedure

### 1. Initialize configuration

Check for `.rdd.yml` in the project root. If absent, copy it from `templates/.rdd.yml`. The `legacy` block may already be filled by the user; if not, ask for it now (just `legacy.stack` and `legacy.source` — enough for the survey step below).

The `target` block stays empty until the end of this skill.

### 2. Run a focused constraints interview

Keep it tight: **5–8 questions**, focused, no open-ended brainstorming. Cover:

- **Why are you migrating?** (cost, vendor lock-in, performance, scale, DX, hiring, compliance) — this is the most important question; the rest follow from it.
- **Team profile.** Solo, small team (2–5), or larger? What languages and frameworks does the team already know well?
- **Deployment constraints.** Cloud provider preferences? On-prem requirements? Data residency / compliance? Existing infra to integrate with?
- **Budget posture.** Penny-pinching (free tiers + VPS), comfortable (managed services), enterprise (custom infra)?
- **Time horizon.** Must ship in 3 months? Willing to invest 1–2 years?
- **Non-negotiables.** Stack components that **must** be kept (existing DB, auth provider, language). Things the user has already decided.
- **Existing pain points** in the legacy that target must fix (cold starts, cost per request, slow tests, deploy friction, etc.).

If the user already knows the target stack, skip ahead to step 5 — the skill still produces `TARGET.md` and `.rdd.yml`, just without the options-comparison phase.

### 3. Survey the legacy at high level

This is **not** the deep inventory `/rdd-map` performs. Read just enough to:

- Identify the legacy architectural style (monolith, function-per-endpoint, services, event-driven).
- Spot integration points that constrain target choice (queue brokers, third-party APIs, webhook receivers, scheduled jobs).
- Estimate domain complexity (CRUD-heavy vs business-rule-heavy vs streaming/stateful) — this favors different stacks.
- Note language(s) used and any FFI / binary dependencies that affect portability.

15–30 minutes of reading at most. The detailed module inventory is `/rdd-map`'s job.

### 4. Generate 2–3 candidate target architectures

For each candidate, document:

- **Stack** — language, framework, runtime, package manager
- **Architecture pattern** — monolithic, modular monolith, microservices, serverless, hybrid
- **Database approach** — shared with legacy (during migration or permanent), migrated, dual-write, fresh schema
- **Auth approach** — keep current provider, replace, hybrid (keep IdP but own the session)
- **Hosting / deploy target** — VPS, PaaS (Railway, Fly, Render), Kubernetes, serverless platform
- **Test framework** — what tests will be written in
- **Strengths** — 2–3 bullets specific to this user's constraints
- **Weaknesses** — 2–3 bullets honest about cost, risk, complexity
- **Estimated migration effort** — relative (S/M/L/XL) compared to other candidates
- **Risk profile** — what's the most likely way this choice ends in regret?

Avoid the cargo-cult default ("just use Next.js"). The right answer depends on the legacy shape and the user's constraints. If a "boring" choice (Postgres + a monolithic framework) fits, propose it.

### 5. Discuss tradeoffs and let the user decide

Present the candidates as a comparison. Make the tradeoffs visible. **Do not decide for the user** — they own the operational consequences for years. Your job is to surface the tradeoffs they may not have considered.

If the user pushes you to "just pick", make a recommendation **with explicit rationale tied to their stated constraints**, but flag that it's your recommendation, not an objective best.

### 6. Document the decision in `TARGET.md`

Create `{artifacts_dir}/TARGET.md` using `templates/TARGET.md`. The file is **a decision record**, not just a description. Every choice is paired with a rationale tied to the user's constraints from step 2.

Sections:

- **Why migrating** — the answer from step 2, in one paragraph
- **Chosen stack** — with rationale
- **Architecture pattern** — with rationale (especially: why this and not the most popular alternative)
- **Database strategy** — shared/migrated/dual-write, with the cutover plan implication
- **Auth strategy** — keep/replace/hybrid, with the migration timing
- **Hosting / deploy** — provider and why
- **Test strategy summary** — framework and the integration-vs-unit posture (referenced by `/rdd-tests` later)
- **Conventions** — naming (camel/snake/kebab), folder structure pattern, error handling style, validation approach, logging, multi-tenancy boundary
- **Cross-cutting concerns** — observability, audit, rate limiting, idempotency
- **Cutover strategy** — strangler/big-bang/dual-write; feature flags? proxy?
- **Out of scope** — architectural decisions explicitly deferred (e.g., "we'll decide event-sourcing later")
- **Open questions** — anything not yet resolved that blocks downstream skills

### 7. Auto-populate `.rdd.yml`

Update the `target` block of `.rdd.yml` programmatically based on `TARGET.md`:

- `target.stack` — short string description
- `target.source` — proposed source dir (ask user to confirm; default like `apps/api/src/` or `services/<svc>/src/`)
- `target.test_framework` — exact framework
- `target.test_strategy` — short summary of the approach from step 6

Do **not** invent values — if `TARGET.md` doesn't have it, ask the user.

### 8. Hand-off

Tell the user:

- Path to `TARGET.md`
- Confirmation that `.rdd.yml` is filled
- Next step: `/rdd-map` will now use the target choice to inform module grouping and migration order

## In-place refactor mode

If the user is doing an **in-place refactor** (same stack, just cleaner code under behavior-locking tests), this skill still runs but is shorter:

- Skip the candidate-generation step (target = legacy stack)
- Focus the interview on **conventions** — naming, folder structure, error handling — that the new code will follow
- `TARGET.md` is mostly the "Conventions" section

## Anti-patterns to avoid

- **Don't pick the trendy default.** "Everyone uses X" is not a justification.
- **Don't decide architecture without input.** This is the user's call. You surface tradeoffs.
- **Don't underestimate boring choices.** Postgres + a monolithic framework solves more problems than people admit.
- **Don't conflate stack with pattern.** "NestJS" is a framework; "modular monolith" is a pattern. They're decided separately.
- **Don't decide things downstream skills should decide.** Module boundaries → `/rdd-map`. Per-module API → `/rdd-spec`. Specific tests → `/rdd-tests`. This skill stays at the architecture level.
- **Don't write `TARGET.md` without rationale.** A doc that says "we use NestJS" without "because [stated constraint]" is useless six months later when someone asks why.

## When to spawn sub-agents

If the legacy stack is one you're less familiar with and the comparison requires up-to-date framework info, use the web-researcher agent (or context7 for specific libraries) to gather current data on the candidate frameworks. Pass it the user's constraints and ask for an honest comparison — strengths, weaknesses, recent breaking changes, community trajectory.
