---
name: rdd-specify-03
description: Use this skill after rdd-map-codebase-02 to capture the business rules of one module (or all modules in batch mode) from legacy code, producing numbered specs that downstream characterization tests will lock. Third step in the RDD pipeline (per-module or batch). Triggers on "spec the X module", "specify X", "spec all modules", "batch spec", "what does X do", "extract business rules". Produces rdd/{module}/SPEC.md per module; batch mode parallelizes across all modules from MAP.md.
---

# rdd-specify-03 — Capture the spec of one module (or all)

You are extracting the **observable behavior** of legacy code into written specs. The spec is the contract that the new implementation must honor. Tests will be designed against this spec in `/rdd-refactor-04`.

## Modes

- **Single module** (default with module name): `/rdd-specify-03 products` — full procedure including user interview for tribal knowledge
- **Batch (sequential autopilot)** (no argument or `all`): `/rdd-specify-03` or `/rdd-specify-03 all` — spec every module from `MAP.md` **one at a time**, recording progress to `BATCH_SPEC.progress.md` so you can resume across sessions if tokens run out or you pause. User interview is **deferred** per module (open questions captured in each `SPEC.md` instead).

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

## Procedure (batch mode — sequential autopilot)

When invoked with `all` or no argument, the skill specs **every module from MAP.md sequentially**, one at a time, while persisting progress to a file so the work survives session interruptions, token exhaustion, and user pauses.

Why sequential and not parallel:

- **Resumable.** If the conversation hits its context limit or you stop mid-run, the next session reads the progress file and continues from the next pending module — none of the completed work is lost.
- **Predictable cost.** One module at a time. No bursts.
- **Failures are isolated.** If module N fails, modules 1..N-1 are still done; you can retry just N.
- **You can pause freely.** Stop after any module by closing the session — progress is durable on disk.

The user interview is **deferred** per module — every unmapped consequence becomes an entry in that module's `SPEC.md` "Open questions" section instead of pausing for tribal knowledge. You batch-resolve open questions later by re-invoking `/rdd-specify-03 <module>` in single mode, or by editing the `SPEC.md` directly.

### B1. Load and validate (once)

Run steps 1 and 2 from the single-module procedure once at the start. The validation applies to all modules (it's about shared inputs: `.rdd.yml`, `TARGET.md`, `MAP.md`).

If validation fails, stop — don't start the autopilot.

### B2. Read MAP.md and extract module list

Parse MAP.md for the module table. Capture per module: name, entry-point count, source path hint, wave. If MAP.md is missing or empty, instruct the user to run `/rdd-map-codebase-02` first.

### B3. Read or create the progress file

`{artifacts_dir}/BATCH_SPEC.progress.md` is the source of truth for resumability. Format:

```markdown
# Batch Spec Progress

**Status:** in_progress | completed
**Started:** YYYY-MM-DD HH:MM
**Last update:** YYYY-MM-DD HH:MM
**Modules:** X/Y completed (Z failed)

## Modules

### 00_foundation
- **Status:** completed | in_progress | pending | failed
- **SPEC.md:** rdd/00_foundation/SPEC.md
- **BRs:** 8  •  **Errors:** 3  •  **Open questions:** 2
- **Notes:** —

### ai-router
- **Status:** pending
- (filled when started)

(... one entry per module from MAP.md, in MAP wave order)
```

**On invocation:**

- If the file does not exist, create it with one entry per module in MAP order, all with `Status: pending`. The user is starting fresh.
- If the file exists, read it. Identify modules with `Status: completed` (skip these), `Status: failed` (skip — user must retry individually), and `Status: pending` or `Status: in_progress` (work to do).
- If a module is `in_progress` from a prior session that died mid-spec, treat as `pending` — the prior partial work may need to be redone. Tell the user: *"Found `<module>` in_progress from a prior session — the SPEC.md may be partial. I'll redo it."*

### B4. Confirm plan with the user (once)

If starting fresh, tell the user:

```
Batch spec — sequential autopilot.
- N modules to spec, in MAP order:
  Wave 0: <modules>
  Wave 1: <modules>
  ...
- Each module: read code, simulate execution, draft SPEC.md, mark complete in BATCH_SPEC.progress.md, continue.
- User interview is deferred — open questions go into each SPEC.md's "Open questions" section.
- I will not stop between modules. If you need to pause, close the session — progress is persisted.

Estimated time: X minutes (rough — bigger modules take longer).
Continue?
```

Wait for confirmation.

If resuming, tell the user:

```
Resuming batch spec — M of N modules already done.
- Completed: <list, count only if many>
- Skipping (previously failed): <list>
- Remaining to spec: <list>

Continue from `<next pending module>`?
```

Wait for confirmation.

### B5. The autopilot loop — one subagent per module

For each pending module in MAP order, the main agent dispatches **one general-purpose subagent** to do the spec work and waits for its completion. The main agent never reads the legacy module code itself in batch mode — that work happens inside the subagent's isolated context.

This is a deliberate context-engineering choice. If the main agent did the work directly, it would accumulate every module's legacy code reading across the whole run, blowing past the context window after 4–6 modules even with prompt caching. With one subagent per module, the main agent only holds: progress-file state, MAP.md (cached), TARGET.md (cached), and per-module result metadata. Each subagent gets a fresh context, does its work, and returns a one-line summary.

**For each pending module, do this in order:**

1. **Mark `in_progress`** in `BATCH_SPEC.progress.md`. Update the timestamp.
2. **Dispatch one general-purpose subagent.** The agent has no prior conversation context — its prompt must be fully self-contained. Use this template (substituting `<module_name>`, source paths, and counts from MAP.md):

   ```
   You are running step "spec one module" as part of an RDD batch autopilot. The main agent has dispatched you with a self-contained prompt — you have no prior conversation context.

   Module: <module_name>
   Module's legacy source files: <comma-separated list or directory hint from MAP.md>
   Module's MAP.md entry-point count: <count>

   Read first (in this order):
   1. <artifacts_dir>/TARGET.md — for chosen conventions, error format, multi-tenancy boundary
   2. <artifacts_dir>/MAP.md — for context on how this module fits in the migration
   3. The plugin's templates/SPEC.md for the artifact structure (path: <plugin_dir>/templates/SPEC.md)

   Then perform these steps from the rdd-specify-03 single-module procedure:

   Step 3 — Deep-read the module's legacy code:
   - Read every entry point and every file it imports within the legacy source
   - Capture entry points (method, path, auth), inputs, outputs per status code, DB ops, external calls, side effects, auth/authz, edge cases handled in code, and bugs the code embraces

   Step 4 — Simulate execution and find unmapped consequences:
   - For each capability, walk through inputs/outputs/related entities/concurrency/failure paths
   - Anything no document addresses → record as an entry in the SPEC.md's "Open questions" section
   - This step is yours to do — do not delegate it further

   Step 6 — Draft SPEC.md incrementally:
   - Write to: <artifacts_dir>/<module_name>/SPEC.md (create the directory if missing)
   - Follow templates/SPEC.md structure exactly: Domain, Use cases, API surface, Business rules (numbered BR-NN), Error Catalog (SCREAMING_SNAKE_CASE codes), Side effects, Out of scope, Open questions, Intentional deviations from legacy
   - Append section by section, not in one giant Write

   Forbidden:
   - Do NOT use AskUserQuestion. The user interview is deferred to a later session.
   - Do NOT skip the Open questions section — every unmapped consequence and every tribal-knowledge gap goes there.
   - Do NOT spec a different module. You are dispatched for <module_name> only.
   - Do NOT modify TARGET.md, MAP.md, .rdd.yml, or any other module's SPEC.md.
   - Do NOT spawn additional sub-agents (no nested delegation).

   For very large modules (>2000 lines), use Glob/Grep/Read efficiently — read entry points first, then dive into imports as needed. Do not read every file in the legacy source if it's not imported by this module.

   Return as your final message a single one-line summary in this exact format:
   <module_name>: <BR_count> BRs, <error_count> errors in catalog, <open_question_count> open questions. SPEC.md at <artifacts_dir>/<module_name>/SPEC.md.

   If you fail (legacy code unreadable, ambiguity blocking work), return instead:
   <module_name>: FAILED — <one-line reason>
   ```

3. **Wait for the subagent to complete.** Do not dispatch the next module until this one returns.
4. **Parse the subagent's one-line summary** and update `BATCH_SPEC.progress.md`:
   - On success: mark `completed`, record BR / Error / Open question counts and SPEC.md path
   - On `FAILED`: mark `failed` with the reason; **continue the loop** — failures are isolated per module
5. **Briefly emit progress to chat** (one line, exactly what the subagent returned, prefixed with status):
   ```
   ✅ <module> — <N> BRs, <M> errors, <K> open questions  (<X>/<Y> complete)
   ```
   or for failures:
   ```
   ❌ <module> — failed: <reason>  (<X>/<Y> complete)
   ```
6. **Continue to the next pending module.** Do NOT stop, do NOT ask for confirmation between modules — the user opted into autopilot.

**Why this design wins on context engineering:**

- **Main agent context stays bounded** — only progress-file state and per-module metadata accumulate. Across 14 modules, that's a few KB of growth, not 14×100k of legacy reading.
- **Each subagent has fresh context** — no contamination from sibling modules, no risk of cross-module bleed in BR numbering or error catalog merging.
- **Cost is predictable per module** — one subagent at a time. No bursts. If a module is a token outlier (huge legacy code), it's bounded to that subagent's run.
- **Resumability is preserved** — the progress file is updated after each subagent returns, so any session interruption resumes cleanly from the next pending module.

### B6. Final report

When the loop finishes:

```
Batch spec complete — <X>/<Y> modules specced (<Z> failed).

| Module          | Status     | BRs | Errors | Open questions |
|-----------------|------------|-----|--------|----------------|
| 00_foundation   | ✅ done    |   8 |      3 |              2 |
| ai-router       | ✅ done    |  12 |      4 |              5 |
| whatsapp-channel| ❌ failed  |   — |      — | (see notes)    |
| ...

Failed modules (retry individually):
- whatsapp-channel — <reason>

Total open questions: <M>

Next steps:
- Resolve open questions per-module — `/rdd-status` shows which need attention
- Run `/rdd-specify-03 <module>` in single mode to interview tribal knowledge, or edit SPEC.md directly
- Retry any failed modules: `/rdd-specify-03 <failed-module>`
- When ready, start `/rdd-refactor-04` on the first module of Wave 0
```

Mark `BATCH_SPEC.progress.md` `Status: completed` (even if some modules failed — the autopilot ran to its end; failures are recorded per module for retry).

### B7. What `/rdd-status` shows

`/rdd-status` reads `BATCH_SPEC.progress.md` if present and reports:

- Setup row: `batch_spec: in progress (M/N)` or `batch_spec: completed`
- The per-module table reflects completed/failed/pending for the spec column

If a session dies mid-batch, `/rdd-status` and the progress file together tell you exactly where to resume.

## Anti-patterns in batch mode

- **Don't run in parallel.** Sequential is the design. Parallel was tried and dropped — it sacrificed resumability for wall-clock time, and resumability matters more for migrations that span days.
- **Don't read the legacy code in the main agent during batch mode.** That defeats the context-engineering benefit of subagent-per-module. The main agent dispatches; the subagent reads.
- **Don't dispatch more than one subagent at a time.** Sequential one-by-one is the design. Two simultaneous subagents = parallel mode in disguise.
- **Don't accumulate per-module summaries in main-agent context as long-form prose.** A single line per module (`<module>: N BRs, M errors, K open questions`) is enough — the durable record is in `BATCH_SPEC.progress.md` and the per-module SPEC.md files.
- **Don't STOP between modules in batch mode.** If the user wants per-module review, they use single mode. Stopping defeats autopilot.
- **Don't use AskUserQuestion in batch mode** (main agent OR subagent). Every question becomes an Open question in the SPEC.md.
- **Don't dispatch the autopilot without user confirmation** of the plan.
- **Don't ask the user N questions.** That's the whole reason batch mode exists.
- **Don't try to merge specs across modules.** Each `SPEC.md` is independent.
- **Don't skip validation pre-flight.** A missing `TARGET.md` decision affects every module — catch it once, not N times.
- **Don't run batch on a project with 1–3 modules.** Single mode is faster end-to-end at that scale (no subagent dispatch overhead).
- **Don't let subagents nest further sub-agents.** Keep the dispatch tree flat: main → one subagent per module → no deeper.
- **Don't retry failed modules inside the autopilot.** Retry is the user's call, individually, after the run completes.
- **Don't delete `BATCH_SPEC.progress.md` until the run is `completed`.** It's the resume anchor.

## When to use which mode

| Situation | Mode |
|-----------|------|
| <4 modules in MAP.md | Single (autopilot overhead isn't worth it) |
| 4+ modules, want hands-off drafts with resumability | **Batch (sequential autopilot)** |
| Resolving open questions on an existing SPEC.md | Single (interview step is the value) |
| Module has high domain complexity / lots of tribal knowledge | Single (interview catches what code doesn't say) |
| Routine CRUD modules | Batch — they're code-readable, few tribal rules |
| Resuming after a session died mid-run | Batch — re-invoke and it picks up where the progress file left off |
