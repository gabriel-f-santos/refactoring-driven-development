---
name: rdd-specify-03
description: Use this skill after rdd-map-codebase-02 to capture the business rules of one module (or all modules in batch mode) from legacy code, producing numbered specs that downstream characterization tests will lock. Third step in the RDD pipeline (per-module or batch). Triggers on "spec the X module", "specify X", "spec all modules", "batch spec", "what does X do", "extract business rules". Produces rdd/{module}/SPEC.md per module; batch mode parallelizes across all modules from MAP.md.
---

# rdd-specify-03 — Capture the spec of one module (or all)

You are extracting the **observable behavior** of legacy code into written specs. The spec is the contract that the new implementation must honor. Tests will be designed against this spec in `/rdd-refactor-04`.

## Modes

- **Single module** (default with module name): `/rdd-specify-03 products` — full procedure including user interview for tribal knowledge
- **Batch** (no argument or `all`): `/rdd-specify-03` or `/rdd-specify-03 all` — parallel-spec all modules from `MAP.md` via subagents; user interview is **deferred** (open questions captured per-module instead)

The single-module procedure is below. **Batch mode** (the long-tail-friendly path for migrations) is at the end. Both end up with the same `SPEC.md` shape per module.

## Procedure (single module)

### 1. Load configuration and prior artifacts

Read in order:

- `.rdd.yml` — for `legacy.source`, `legacy.stack`, `artifacts_dir`, and `conventions.business_rule_prefix`
- `{artifacts_dir}/TARGET.md` — for chosen conventions, error format, multi-tenancy boundary
- `{artifacts_dir}/MAP.md` — to confirm the module exists and read prior notes about it

If the user invoked the skill without specifying a module name, list modules from `MAP.md` and ask which.

### 2. Validate inputs (pre-flight)

Before reading legacy code, check for problems that would produce a flawed spec:

**Inputs present:**

- `MAP.md` lists this module with non-empty entry-point count
- `TARGET.md` Decisions Summary is fully filled (no `<pending>`)
- `target.test_framework` is set in `.rdd.yml`

**Cross-document consistency:**

- The error format chosen in `TARGET.md` is what BRs will reference; if missing, escalate
- Multi-tenancy boundary from `TARGET.md` matches what the legacy code actually enforces (read 2-3 entry-point files quickly to verify before going deep)
- Capabilities of this module in `MAP.md` don't contradict each other or contradict another module's claims (e.g., two modules both claim ownership of the same DB table)

**If any issue is found:** stop. Surface to the user. Wait for resolution before proceeding.

**If no issues:** proceed.

### 3. Read the module's legacy code

For the chosen module, read **every entry point and every file it imports** within the legacy source. This is the deep-read step — `/rdd-map-codebase-02` only sampled.

Capture from the code:

- **Entry points** — HTTP routes, queue consumers, scheduled jobs, webhook receivers (method, path, auth requirements)
- **Inputs** — request body shape, query params, headers, JWT claims used
- **Outputs** — response shape per status code, including error responses
- **Database operations** — tables read, tables written, transactions, locking
- **External calls** — APIs hit, with payload shape and idempotency assumptions
- **Side effects** — emails sent, push notifications, audit logs, analytics events, queue messages emitted
- **Auth/authorization** — required roles, permission checks, multi-tenant filters
- **Edge cases handled in code** — explicit `if`s for special inputs, retries, fallback logic
- **Bugs the code seems to embrace** — non-obvious behavior that callers may rely on. **Note these explicitly.** Bug-for-bug parity is the default.

### 4. Simulate execution and find unmapped consequences

This step is what separates a serious spec from prose. Code reading is static. **Mentally simulate** each capability's execution and probe for consequences no document currently addresses. For every capability:

**a) Inputs and their edge cases.** Walk realistic input variance:

- What if the input is empty / null / missing?
- What if it's at the boundary (0, -1, max int, very long string)?
- What if it contains characters that get stripped or transformed (unicode, accents, leading/trailing whitespace, special chars)?
- What if two requests provide the same "unique" input simultaneously?

Example: *"Channel handle derived from email prefix — what if two users share the same prefix? What if the prefix is one character? 200 characters? Only special chars that get stripped?"*

**b) Outputs and what they touch.** Trace every derived or generated value:

- Can outputs collide with each other (uniqueness violations)?
- Can outputs overflow downstream constraints (length limits, format requirements)?
- Are downstream consumers of these outputs explicitly handled?

**c) Related entities and side effects.** When the operation creates, modifies, or deletes data:

- What entities are touched directly or transitively?
- Are cascading effects specified (e.g., delete user → what happens to their videos, comments, jobs)?
- Are state transitions in related entities clear (e.g., change email → does derived channel handle update, or is it frozen)?

**d) Concurrency and repeated execution.**

- Two users / requests doing the same op simultaneously — race conditions? duplicate records? inconsistent states?
- Same request retried (network blip, client retry) — idempotent? double-charge?
- Webhook redelivery (Stripe, etc.) — addressed?

**e) Failure paths.** For each step that can fail:

- Is there a defined rollback?
- Error response shape and code?
- Cleanup of partial state? (e.g., "registration creates user + channel — what if channel creation fails after user row inserted? Is it transactional?")

**Output of this step:** a list of unmapped consequences. Each is either:

- **Resolved by reading more code** (the code does handle it, you just hadn't found it yet) — fold into a BR
- **Implicit but unaddressed** (legacy code happens to work but no rule states it) — escalate as a question for the user
- **Genuine gap** (legacy bug or missing handling) — escalate; user decides whether to preserve (parity) or fix (intentional deviation)

Don't enumerate every possible edge case exhaustively — simulate realistic execution paths and flag what no document currently addresses.

### 5. Interview the user (mandatory)

Code never tells the whole story. Ask about:

- **Tribal rules** — undocumented constraints (e.g., "we never charge twice on the same day", "company X gets 5% extra discount")
- **Recent incidents** — bugs fixed in this module recently; the test plan must cover them
- **Intentional behavior changes** — does the user want to keep a legacy bug or fix it during migration? **Default: keep it.** Deviations are marked.
- **Out-of-scope items** — explicitly list so the migration doesn't grow
- **Unmapped consequences from step 4** — present each unresolved item; user decides

Keep tight. 5–10 focused questions, not open-ended survey.

### 6. Draft `SPEC.md` incrementally

Create `{artifacts_dir}/{module}/SPEC.md` using `templates/SPEC.md`. **Write incrementally** — don't try to one-shot a long file. Append section by section: Domain → Use cases → API surface → Business rules → Error Catalog → Side effects → Out of scope → Open questions → Intentional deviations.

Sections:

#### Domain

- Entities (key fields, invariants)
- Aggregates and ownership
- Multi-tenancy boundary (e.g., `company_id`) — must match what `TARGET.md` declares

#### Use cases

- One per business verb ("Create sale", "Cancel appointment", "Refund commission")
- Each: actor, preconditions, steps, postconditions, observable side effects

#### API surface

- Table of endpoints: method, path, auth, request, response, status codes
- Idempotency key behavior
- Pagination/sorting defaults
- **Follow REST conventions from `templates/SPEC.md`**: 200 returns data, 201 returns created resource, 204 returns no body for action endpoints

#### Business rules

Numbered list using `{conventions.business_rule_prefix}` (default `BR-01`...). Each rule must be:

- **Testable** — asserts something observable
- **Sourced** — cites `(code: path:line)`, `(user-confirmed)`, or `(legacy bug, intentional parity)`
- **Specific** — one rule per assertable fact

**GOOD business rules:**

- ✅ "BR-04 — Sale total equals sum of line items minus discounts, rounded half-up to 2 decimals. `(code: legacy/sales.ts:128)`"
- ✅ "BR-09 — A customer cannot have two appointments overlapping by ≥1 minute on the same employee's calendar. `(code: legacy/appointments.ts:64; user-confirmed)`"
- ✅ "BR-12 — Stripe webhook with same `event.id` is idempotent: second delivery returns 200 without re-applying side effects. `(code: legacy/webhooks/stripe.ts:22)`"

**BAD business rules** (rewrite or drop):

- ❌ "BR-03 — The system should be fast." *(not testable)*
- ❌ "BR-07 — Use Postgres for storage." *(implementation, not behavior)*
- ❌ "BR-11 — The code should be clean." *(vague, not behavior)*

#### Error Catalog

For every domain-specific error this module produces:

- **Code** — application-level identifier in `SCREAMING_SNAKE_CASE` (e.g., `EMAIL_ALREADY_EXISTS`, `OVERLAP_DETECTED`). Globally unique within the subproject.
- **HTTP** — status code returned with this error
- **Message** — human-readable message in the response body's `message` field
- **Trigger** — specific operation + branch that causes this error

**Include:** domain-specific errors (business rules, state violations), auth/authz errors with phase-specific semantics.

**Don't include:** generic framework errors (500, 404 unmatched routes), generic validation errors that the validation lib already returns. Reference these in API Contracts as "400 validation error" without a code.

BRs reference these codes by name. Example BR: *"BR-08 — Empty product name returns 422 with `NAME_REQUIRED`. `(legacy bug — frontend depends on this exact code, intentional parity)`"*

#### Side effects

Table per side effect: trigger, side effect, payload shape.

#### Out of scope

Explicit list of what this module does **not** do.

#### Open questions

Things you couldn't determine and need resolution before `/rdd-refactor-04`.

#### Intentional deviations from legacy

Empty by default — parity is the rule. Filled only when the user explicitly approves a deviation.

### 7. Resolve `[DECIDE]` markers

If during drafting you couldn't pin a detail, insert an inline `[DECIDE: option A | option B — context]` marker and continue. After the draft is complete, present each marker to the user via `AskUserQuestion` (up to 4 per call; sequential calls if more). Apply answers with targeted `Edit`s.

### 8. Hand-off

Tell the user:

- Path to `SPEC.md`
- Number of business rules captured
- Number of errors in Error Catalog
- Any open questions still pending
- Next step: `/rdd-refactor-04 {module}`

## Quality bar for business rules

A good BR is testable, sourced, and specific. Each maps to at least one upcoming test in `/rdd-refactor-04`. If a BR can't be turned into a test, it's not behavior — rewrite or drop it.

## Anti-patterns to avoid

- **Don't describe implementation** — internal class structure, ORM choice, framework decorators. The spec is about what callers and the database see.
- **Don't smooth over bugs** — if legacy returns 200 instead of 422 for invalid input, write that down. Migration locks current behavior; bug fixes are separate, marked changes.
- **Don't make rules up.** Every rule traces to code, user statement, or explicit decision. Speculation goes to "Open questions".
- **Don't skip the user interview.** Code-only specs miss tribal knowledge.
- **Don't skip step 4 (simulation).** Static reading misses concurrency and edge cases the code happens to handle but no document captures.
- **Don't write Error Catalog from imagination.** Codes come from grep'ing the legacy code for what it actually emits.

## When to spawn sub-agents (single mode)

For modules with >20 entry points or >2000 lines of legacy code, use the Explore agent (very-thorough) to do the deep code read and return a structured summary. Then synthesize the spec yourself. Do **not** delegate the simulation step (4) — it requires the model's own reasoning over the structured data.

---

## Procedure (batch mode)

When invoked with `all` or no argument, the skill specs **every module from MAP.md in parallel** via subagents. This is the productive path for migrations: one phase, many specs, hands-off until you review the open questions.

The user interview is **deferred** — each subagent records unmapped consequences in its `SPEC.md`'s "Open questions" section instead of pausing for tribal knowledge. You batch-resolve open questions later (re-invoke `/rdd-specify-03 <module>` in single mode, or edit `SPEC.md` directly).

### B1. Load and validate (once)

Run steps 1 and 2 from the single-module procedure once at the start. The validation applies to all modules (it's about shared inputs: `.rdd.yml`, `TARGET.md`, `MAP.md`).

If validation fails, stop — don't dispatch any subagents.

### B2. Read MAP.md and extract module list

Parse MAP.md for the module table. Capture for each module:

- Name
- Entry-point count (informs subagent thoroughness)
- Files / source path hint
- Wave (informs batching order)

If `MAP.md` is missing or has zero modules, instruct the user to run `/rdd-map-codebase-02` first.

### B3. Confirm batch with the user

Tell the user:

```
Batch mode — about to spec N modules in parallel.
- Wave 0: <modules>
- Wave 1: <modules>
- ...
User interview is deferred. Each module's open questions go into its SPEC.md's "Open questions" section. Estimated time: X minutes (depends on module sizes). Continue?
```

Wait for confirmation. **Do not dispatch agents without explicit yes.** This is a paid, parallel operation — burning compute on misalignment is wasteful.

### B4. Dispatch subagents in waves

For each MAP wave (or in batches of 5–7 if a wave is larger):

- Spawn **one general-purpose agent per module**, all in a **single message** so they run concurrently.
- Each agent prompt is self-contained (the agent has no conversation context). Include:
  - The module name and its files / source path from MAP.md
  - Path to `TARGET.md` and `MAP.md` (so the agent can read them)
  - Path to `templates/SPEC.md` for structure
  - Where to write: `{artifacts_dir}/{module}/SPEC.md`
  - Explicit instruction: **"Skip the user interview step. Capture every unmapped consequence and tribal-knowledge gap in the SPEC.md's 'Open questions' section instead. Do NOT use AskUserQuestion."**
  - Steps 3, 4, and 6 from the single-module procedure (read code, simulate execution, draft SPEC.md). Skip step 5 (interview) and step 7 (`[DECIDE]` markers — those become Open questions in batch mode).
  - Return value: path to written SPEC.md, BR count, Error Catalog count, Open question count
- Wait for all agents in the wave to complete before dispatching the next wave.

### B5. Aggregate and report

After all waves complete:

```
Batch spec complete — N modules specced.

| Module          | BRs | Errors | Open questions |
|-----------------|-----|--------|----------------|
| 00_foundation   |   8 |      3 |              2 |
| ai-router       |  12 |      4 |              5 |
| products        |  18 |      6 |              1 |
| ...

Total open questions: M

Next steps:
- Review open questions per-module — `/rdd-status` shows which modules need attention
- Resolve them by running `/rdd-specify-03 <module>` in single mode (fills gaps via interview), or edit SPEC.md directly
- When ready, start /rdd-refactor-04 on the first module of Wave 0
```

### B6. Don't write a global progress file

Each `SPEC.md` is its own durable artifact. `/rdd-status` infers per-module state from disk. Don't create a `BATCH.progress.md` — there's no execution sequence to resume in batch mode (subagents either complete or fail; failures are reported and the user re-runs the failing module individually).

## Anti-patterns in batch mode

- **Don't dispatch without user confirmation** of the wave plan. Parallel agents are paid; surprise the user and you waste budget.
- **Don't ask the user 14 questions.** That's the whole reason batch mode exists. Capture as Open questions, resolve later.
- **Don't try to merge specs across modules** during aggregation. Each module's SPEC.md is independent.
- **Don't skip validation pre-flight.** A missing `TARGET.md` decision affects every module — catch it once, not 14 times.
- **Don't run batch on a project with 1–3 modules.** Single-module mode is faster end-to-end at that scale (no subagent overhead).
- **Don't delegate the simulation step (step 4 from single mode).** Each subagent should run it for its own module. The reasoning is the value.
- **Don't dispatch a wave that exceeds 10 parallel agents.** Split into smaller batches; orchestrate sequentially across batches.

## When to use which mode

| Situation | Mode |
|-----------|------|
| <4 modules in MAP.md | Single (overhead of batch isn't worth it) |
| 4+ modules, want fast turnaround on drafts | **Batch** |
| Resolving open questions on an existing SPEC.md | Single (the interview step is the value) |
| Module has high domain complexity / lots of tribal knowledge | Single (interview catches what code doesn't say) |
| Routine CRUD modules | Batch (code-readable, few tribal rules) |
