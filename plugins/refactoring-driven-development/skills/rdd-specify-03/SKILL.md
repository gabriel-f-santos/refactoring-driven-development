---
name: rdd-specify-03
description: Use this skill after rdd-map-codebase-02 to capture the business rules of one module (or all modules in batch mode) from legacy code, producing numbered specs that downstream characterization tests will lock. Third step in the RDD pipeline (per-module or batch). Triggers on "spec the X module", "specify X", "spec all modules", "batch spec", "what does X do", "extract business rules". Produces rdd/<Seq>_<module>/spec/SPEC.md and rdd/<Seq>_<module>/PROGRESS.md per module (e.g., rdd/001_foundation/spec/SPEC.md); batch mode iterates across all modules from MAP.md in ascending Seq order.
---

# rdd-specify-03 — Capture the spec of one module (or all)

You are extracting the **observable behavior** of legacy code into written specs. The spec is the contract that the new implementation must honor. Tests will be designed against this spec in `/rdd-refactor-04`.

This skill is **autonomous**: the legacy code already exists, so the source of truth is reading and simulating it — not interviewing the user. Anything you can't determine from code is captured as an entry under "Open questions" in `SPEC.md`. The user resolves those later by editing `SPEC.md` directly or re-running `/rdd-specify-03 <module>` after they've answered. Architectural decisions live in `/rdd-specify-01` (`TARGET.md`) — that's the only interview step in the pipeline.

## Modes

Both modes produce the same `SPEC.md` shape per module. They differ only in orchestration:

- **Single module** (`/rdd-specify-03 products`) — spec one module. Use when iterating on a specific module's spec, or after the user has answered Open questions and you want to re-spec with the new info.
- **Batch (sequential autopilot)** (no argument or `all`): `/rdd-specify-03` or `/rdd-specify-03 all` — spec every module from `MAP.md` **one at a time**, recording progress to `BATCH_SPEC.progress.md` so you can resume across sessions if tokens run out or you pause.

The single-module procedure is below. **Batch mode** (the long-tail-friendly path for migrations) is at the end.

## Procedure (single module)

### 1. Load configuration and prior artifacts

Read in order:

- `.rdd.yml` — for `legacy.source`, `legacy.stack`, `artifacts_dir`, and `conventions.business_rule_prefix`
- `{artifacts_dir}/TARGET.md` — for chosen conventions, error format, multi-tenancy boundary
- `{artifacts_dir}/MAP.md` — to confirm the module exists, read prior notes about it, and look up its **`Seq`** (sequence number from the Modules table)

If the user invoked the skill without a module name, fall through to **batch mode** (spec all modules sequentially in Seq order). Don't stop to ask — batch mode is the documented default for no-arg invocation.

### 1.5. Resolve module directory

Module artifacts live at `{artifacts_dir}/<Seq>_<module>/` (e.g., `rdd/001_foundation/`). This skill is responsible for **creating that directory** when it doesn't exist yet — `/rdd-map-codebase-02` only records `Seq` in `MAP.md`.

**Resolution rules:**

1. The user can invoke with any of three forms — all must resolve to the same directory:
   - Bare module name — `/rdd-specify-03 reports`
   - Pre-prefixed — `/rdd-specify-03 005_reports`
   - Seq only — `/rdd-specify-03 005` (looks up which module has Seq=005 in MAP)
2. Parse the input:
   - Matches `^[0-9]{3}$` → Seq-only. Look up MAP.md's Modules table for the row with this `Seq`; the module name comes from there.
   - Matches `^[0-9]{3}_(.+)$` → Pre-prefixed. The first 3 digits are the `Seq`; the suffix is the module name.
   - Otherwise → bare name. Look up MAP.md's Modules table for the row with this module name; `Seq` comes from there.
   If the lookup fails (Seq or name not in MAP), stop — instruct the user to re-run `/rdd-map-codebase-02`.
3. **Glob for an existing directory** matching `{artifacts_dir}/[0-9][0-9][0-9]_<module>/`. Three cases:
   - **Exactly one match** — use it. If the directory's `Seq` prefix differs from MAP's `Seq`, surface the discrepancy and ask the user whether to rename (`git mv`) or update MAP.
   - **No match** — directory doesn't exist yet. Create `{artifacts_dir}/<Seq>_<module>/` using the `Seq` from MAP.
   - **Multiple matches** — should not happen; a module name is unique. Stop and surface the conflict for the user to resolve.
4. The resolved path becomes `module_dir` for the rest of the procedure. Anywhere this skill previously wrote to `{artifacts_dir}/{module}/SPEC.md`, it now writes to `{module_dir}/SPEC.md`.

**Backward compatibility:** if MAP.md has no `Seq` column (legacy MAP from a prior plugin version), use the bare module name (`{artifacts_dir}/{module}/`) and emit a one-line warning suggesting the user re-run `/rdd-map-codebase-02` to assign sequence numbers.

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

- **Resolved by reading more code** (the code does handle it, you just hadn't found it yet) — fold into a BR with the code citation
- **Implicit but unaddressed** (legacy code happens to work but no rule states it) — record as an entry in the **Open questions** section of `SPEC.md`, written so the user can answer in one line
- **Genuine gap** (legacy bug or missing handling) — record as an entry in **Open questions**; default assumption is parity (legacy is the truth) and you note that as the recommendation for that question

Don't enumerate every possible edge case exhaustively — simulate realistic execution paths and flag what no document currently addresses.

### 5. Draft `SPEC.md` incrementally

Create `{module_dir}/spec/SPEC.md` (the `module_dir` resolved in step 1.5, plus a `spec/` subdirectory — e.g., `{artifacts_dir}/005_reports/spec/SPEC.md`) using `templates/SPEC.md`. **Write incrementally** — don't try to one-shot a long file. Append section by section: Domain → Use cases → API surface → Business rules → Error Catalog → Side effects → Out of scope → Open questions → Intentional deviations.

The module directory follows this layout (full structure documented in `templates/PROGRESS.md`):

```
{module_dir}/
├── PROGRESS.md              ← created in step 7 below
├── spec/
│   ├── SPEC.md              ← this step writes here
│   └── TESTS.md             ← created later by /rdd-refactor-04
├── tasks/                   ← created later by /rdd-refactor-04 (port phase)
└── improve/                 ← created later by /rdd-improve-05 (lazy, optional)
```

Create the `spec/` subdirectory if it doesn't exist.

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

### 6. Resolve gaps without asking

When you can't pin a detail during drafting, **do not** stop and ask the user. Two paths:

- **Pick the parity-default** and keep going. If the legacy code does X under condition Y, write the BR as "BR-NN — under Y, system does X. `(code: path:line)`". Default for ambiguous behavior: whatever the legacy currently does is the rule.
- **If even the parity behavior is ambiguous** (e.g., the code path is unreachable, or two branches contradict), record an entry under **Open questions** describing the ambiguity, the options, and your recommended default. The user resolves later by editing `SPEC.md` or re-running this skill.

This is the autonomy contract: the skill produces a complete spec from code-as-truth + simulation, with gaps surfaced explicitly under Open questions. The user reviews `SPEC.md`, edits inline, and moves on. No mid-spec dialog.

### 7. Initialize `PROGRESS.md` and hand off

After `spec/SPEC.md` is fully drafted, create `{module_dir}/PROGRESS.md` from `templates/PROGRESS.md` if it doesn't exist yet. Fill in:

- Module Seq + name in the title
- Set the `spec` row of the Pipeline table to `✅` with detail (BR count, error count, open questions count)
- Capture upstream gate state: read MAP.md's `Depends on` column; for each upstream, write a row with status (`✅ completed` / `🔄 in_progress` / `⬜ pending` / `⏭ skipped`) by checking `{artifacts_dir}/<upstream_Seq>_<upstream>/PROGRESS.md` (or fall back to `REFACTOR.progress.md` for backward compat)
- Append one history line: `<timestamp> — /rdd-specify-03 — spec phase completed: N BRs, M errors, K open questions`

If `PROGRESS.md` already exists (re-run on a module that was previously specced), don't overwrite — only update the spec row and append the new history line.

**Validate dependency readiness:** read each upstream's `PROGRESS.md`. If any upstream's port is not `completed` and the module isn't `skipped:` in `.rdd.yml`, the hand-off message warns that `/rdd-refactor-04` will block at the upstream gate.

Tell the user:

- Path to `spec/SPEC.md` (`{module_dir}/spec/SPEC.md`)
- Path to `PROGRESS.md` (`{module_dir}/PROGRESS.md`)
- Number of business rules captured
- Number of errors in Error Catalog
- Any open questions still pending
- **If all upstream deps have completed ports:** Next step: `/rdd-refactor-04 {module}` — ready to start.
- **If some upstream deps are not yet ported:** Next step: complete `/rdd-refactor-04 <upstream>` first. List the pending upstreams (`<Seq>_<name>`) so the user can plan. The Phase 0 gate in `/rdd-refactor-04 {module}` will enforce this — bypass requires marking those upstreams `skipped:` in `.rdd.yml`.

If the resolved `Seq` for this module conflicts with the dependencies' `Seq` (e.g., this module is `003` but depends on `005`), also warn — the topological order is wrong and the user should re-run `/rdd-map-codebase-02` to renumber.

## Quality bar for business rules

A good BR is testable, sourced, and specific. Each maps to at least one upcoming test in `/rdd-refactor-04`. If a BR can't be turned into a test, it's not behavior — rewrite or drop it.

## Anti-patterns to avoid

- **Don't describe implementation** — internal class structure, ORM choice, framework decorators. The spec is about what callers and the database see.
- **Don't smooth over bugs** — if legacy returns 200 instead of 422 for invalid input, write that down. Migration locks current behavior; bug fixes are separate, marked changes.
- **Don't make rules up.** Every rule traces to code or to an explicit Open question. Speculation goes to "Open questions" with a recommended default.
- **Don't ask the user mid-spec.** Code is the source of truth; simulation finds gaps; gaps go to Open questions. The user resolves by editing `SPEC.md` after, not during. The single interactive step in the whole pipeline is `/rdd-specify-01` (architecture).
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

Parse MAP.md for the module table. Capture per module: **`Seq`**, name, entry-point count, source path hint, wave. If MAP.md is missing or empty, instruct the user to run `/rdd-map-codebase-02` first. If MAP.md has no `Seq` column (legacy from prior plugin version), instruct the user to re-run `/rdd-map-codebase-02` to add sequence numbers — batch mode requires `Seq` to know the iteration order.

**Iterate modules in ascending `Seq` order** — the lowest `Seq` (e.g., `001_<module>`) is processed first. This guarantees that when a module's spec is being written, its upstreams have either already been specced earlier in the same batch run or were declared dependencies in MAP.

### B3. Read or create the progress file

`{artifacts_dir}/BATCH_SPEC.progress.md` is the source of truth for resumability. Format:

```markdown
# Batch Spec Progress

**Status:** in_progress | completed
**Started:** YYYY-MM-DD HH:MM
**Last update:** YYYY-MM-DD HH:MM
**Modules:** X/Y completed (Z failed)

## Modules

### 001_foundation
- **Status:** completed | in_progress | pending | failed
- **SPEC.md:** rdd/001_foundation/spec/SPEC.md
- **PROGRESS.md:** rdd/001_foundation/PROGRESS.md
- **BRs:** 8  •  **Errors:** 3  •  **Open questions:** 2
- **Notes:** —

### 002_auth
- **Status:** pending
- (filled when started)

(... one entry per module from MAP.md, in ascending Seq order — heading uses <Seq>_<module>)
```

**On invocation:**

- If the file does not exist, create it with one entry per module in **ascending `Seq` order**, all with `Status: pending`. Each module heading uses `<Seq>_<module>` (e.g., `### 001_foundation`). The user is starting fresh.
- If the file exists, read it. Identify modules with `Status: completed` (skip these), `Status: failed` (skip — user must retry individually), and `Status: pending` or `Status: in_progress` (work to do).
- If a module is `in_progress` from a prior session that died mid-spec, treat as `pending` — the prior partial work may need to be redone. Tell the user: *"Found `<module>` in_progress from a prior session — the SPEC.md may be partial. I'll redo it."*

### B4. Announce the plan and proceed (no confirmation)

Print a single status line so the user knows what's happening, then start the loop. Do not block on confirmation — the user invoked the skill, and progress is durable on disk (`BATCH_SPEC.progress.md`) so they can pause by closing the session.

If starting fresh:

```
Batch spec — N modules in Seq order: 001_foundation, 002_auth, 003_<...>, ...
Each module: read code, simulate, draft SPEC.md with Open questions for unresolved items. No prompts between modules.
```

If resuming:

```
Resuming batch spec from <next pending module> — M of N modules already done.
```

Then proceed directly to B5.

### B5. The autopilot loop — one subagent per module

For each pending module in MAP order, the main agent dispatches **one general-purpose subagent** to do the spec work and waits for its completion. The main agent never reads the legacy module code itself in batch mode — that work happens inside the subagent's isolated context.

This is a deliberate context-engineering choice. If the main agent did the work directly, it would accumulate every module's legacy code reading across the whole run, blowing past the context window after 4–6 modules even with prompt caching. With one subagent per module, the main agent only holds: progress-file state, MAP.md (cached), TARGET.md (cached), and per-module result metadata. Each subagent gets a fresh context, does its work, and returns a one-line summary.

**For each pending module, do this in order:**

1. **Mark `in_progress`** in `BATCH_SPEC.progress.md`. Update the timestamp.
2. **Dispatch one general-purpose subagent.** The agent has no prior conversation context — its prompt must be fully self-contained. Use this template (substituting `<module_name>`, `<seq>` (3-digit, zero-padded), source paths, and counts from MAP.md):

   ```
   You are running step "spec one module" as part of an RDD batch autopilot. The main agent has dispatched you with a self-contained prompt — you have no prior conversation context.

   Module: <module_name>
   Sequence number (Seq): <seq>            # e.g., 001
   Module directory: <artifacts_dir>/<seq>_<module_name>/    # e.g., rdd/001_foundation/
   Module's legacy source files: <comma-separated list or directory hint from MAP.md>
   Module's MAP.md entry-point count: <count>

   Read first (in this order):
   1. <artifacts_dir>/TARGET.md — for chosen conventions, error format, multi-tenancy boundary
   2. <artifacts_dir>/MAP.md — for context on how this module fits in the migration; note the Depends on column for this module
   3. The plugin's templates/SPEC.md for the artifact structure (path: <plugin_dir>/templates/SPEC.md)

   Then perform these steps from the rdd-specify-03 single-module procedure:

   Step 3 — Deep-read the module's legacy code:
   - Read every entry point and every file it imports within the legacy source
   - Capture entry points (method, path, auth), inputs, outputs per status code, DB ops, external calls, side effects, auth/authz, edge cases handled in code, and bugs the code embraces

   Step 4 — Simulate execution and find unmapped consequences:
   - For each capability, walk through inputs/outputs/related entities/concurrency/failure paths
   - Anything no document addresses → record as an entry in the SPEC.md's "Open questions" section
   - This step is yours to do — do not delegate it further

   Step 5 — Draft SPEC.md incrementally:
   - Write to: <artifacts_dir>/<seq>_<module_name>/spec/SPEC.md (create the spec/ subdirectory if missing)
   - Follow templates/SPEC.md structure exactly: Domain, Use cases, API surface, Business rules (numbered BR-NN), Error Catalog (SCREAMING_SNAKE_CASE codes), Side effects, Out of scope, Open questions, Intentional deviations from legacy
   - Append section by section, not in one giant Write

   Step 6 — Initialize PROGRESS.md:
   - If <artifacts_dir>/<seq>_<module_name>/PROGRESS.md doesn't exist, create it from templates/PROGRESS.md
   - Set the spec phase row to ✅ with BR/error/OQ counts
   - Capture upstream gate from MAP.md's Depends on column for this module
   - Append a history line with timestamp + skill name + event

   Forbidden:
   - Do NOT use AskUserQuestion. This skill is autonomous — gaps go under Open questions in SPEC.md, not into a dialog.
   - Do NOT skip the Open questions section — every unmapped consequence and every tribal-knowledge gap goes there with a recommended default (parity-first).
   - Do NOT spec a different module. You are dispatched for <module_name> only.
   - Do NOT modify TARGET.md, MAP.md, .rdd.yml, or any other module's SPEC.md.
   - Do NOT spawn additional sub-agents (no nested delegation).

   For very large modules (>2000 lines), use Glob/Grep/Read efficiently — read entry points first, then dive into imports as needed. Do not read every file in the legacy source if it's not imported by this module.

   Return as your final message a single one-line summary in this exact format:
   <seq>_<module_name>: <BR_count> BRs, <error_count> errors in catalog, <open_question_count> open questions. spec/SPEC.md + PROGRESS.md at <artifacts_dir>/<seq>_<module_name>/.

   If you fail (legacy code unreadable, ambiguity blocking work), return instead:
   <seq>_<module_name>: FAILED — <one-line reason>
   ```

3. **Wait for the subagent to complete.** Do not dispatch the next module until this one returns.
4. **Parse the subagent's one-line summary** and update `BATCH_SPEC.progress.md`:
   - On success: mark `completed`, record BR / Error / Open question counts and SPEC.md path
   - On `FAILED`: mark `failed` with the reason; **continue the loop** — failures are isolated per module
5. **Briefly emit progress to chat** (one line, exactly what the subagent returned, prefixed with status):
   ```
   ✅ <seq>_<module> — <N> BRs, <M> errors, <K> open questions  (<X>/<Y> complete)
   ```
   or for failures:
   ```
   ❌ <seq>_<module> — failed: <reason>  (<X>/<Y> complete)
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

| Module               | Status     | BRs | Errors | Open questions |
|----------------------|------------|-----|--------|----------------|
| 001_foundation       | ✅ done    |   8 |      3 |              2 |
| 002_auth        | ✅ done    |  12 |      4 |              5 |
| 003_whatsapp-channel | ❌ failed  |   — |      — | (see notes)    |
| ...

Failed modules (retry individually):
- 003_whatsapp-channel — <reason>

Total open questions: <M>

Next steps:
- Review SPEC.md per-module — Open questions list whatever the code couldn't resolve, with a parity-first recommended default for each
- Edit SPEC.md to answer open questions inline, or re-run `/rdd-specify-03 <module>` after the legacy code changes
- Retry any failed modules: `/rdd-specify-03 <failed-module>`
- When ready, start `/rdd-refactor-04` on the lowest-Seq module (Phase 0 enforces upstream-port-completed gates)
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
- **Don't ask the user N questions.** Single mode and batch mode are both autonomous — Open questions in `SPEC.md` is the only channel for unresolved items.
- **Don't try to merge specs across modules.** Each `SPEC.md` is independent.
- **Don't skip validation pre-flight.** A missing `TARGET.md` decision affects every module — catch it once, not N times.
- **Don't run batch on a project with 1–3 modules.** Single mode is faster end-to-end at that scale (no subagent dispatch overhead).
- **Don't let subagents nest further sub-agents.** Keep the dispatch tree flat: main → one subagent per module → no deeper.
- **Don't retry failed modules inside the autopilot.** Retry is the user's call, individually, after the run completes.
- **Don't delete `BATCH_SPEC.progress.md` until the run is `completed`.** It's the resume anchor.

## When to use which mode

| Situation | Mode |
|-----------|------|
| <4 modules in MAP.md | Single (per-module invocation; autopilot orchestration overhead isn't worth it) |
| 4+ modules, fresh start | **Batch (sequential autopilot)** |
| Re-spec a single module after the user edited Open questions or legacy changed | Single |
| Resuming after a session died mid-batch | Batch — re-invoke and it picks up where the progress file left off |
