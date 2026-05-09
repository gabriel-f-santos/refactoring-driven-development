---
name: rdd-spec
description: Use this skill after rdd-map to capture the business rules of one module from its legacy code, producing a numbered spec that downstream characterization tests will lock. Triggers on "spec the X module", "what does X do", "extract business rules from X". Produces rdd/{module}/SPEC.md.
---

# rdd-spec — Capture the spec of one module

You are extracting the **observable behavior** of one legacy module into a written spec. The spec is the contract that the new implementation must honor. Tests will be designed against this spec in `/rdd-tests`.

## Procedure

### 1. Load configuration and prior artifacts

Read in order:

- `.rdd.yml` — for `legacy.source`, `legacy.stack`, `artifacts_dir`, and `conventions.business_rule_prefix`.
- `{artifacts_dir}/MAP.md` — to confirm the module exists and to read your own prior notes about it.

If the user invoked the skill without specifying a module name, list the modules from `MAP.md` and ask which one.

### 2. Read the module's legacy code

For the chosen module, read **every entry point and every file it imports** within the legacy source. This is the deep-read step — `/rdd-map` only sampled.

Capture from the code:

- **Entry points** — HTTP routes, queue consumers, scheduled jobs, webhook receivers. Note method, path, auth requirements.
- **Inputs** — request body shape, query params, headers, JWT claims used.
- **Outputs** — response shape per status code, including error responses.
- **Database operations** — tables read, tables written, transactions, locking.
- **External calls** — APIs hit, with payload shape and idempotency assumptions.
- **Side effects** — emails sent, push notifications, audit logs, analytics events, queue messages emitted.
- **Auth/authorization** — required roles, permission checks, multi-tenant filters.
- **Edge cases handled in code** — explicit `if`s for special inputs, retries, fallback logic.
- **Bugs the code seems to embrace** — non-obvious behavior that callers may rely on. **Note these explicitly.** Bug-for-bug parity is the default.

### 3. Interview the user (mandatory)

The code never tells the whole story. Ask the user about:

- **Tribal rules** — undocumented constraints ("we never charge twice on the same day", "company X gets 5% extra discount").
- **Recent incidents** — bugs fixed in this module recently; the test plan must cover them.
- **Intentional behavior changes** — does the user want to keep the legacy bug or fix it during migration? **Default to keeping it**; deviations must be marked in the spec.
- **Out-of-scope items** — explicitly listed so the migration doesn't grow.

Keep the interview tight. 5–10 focused questions, not an open-ended survey.

### 4. Write `SPEC.md`

Create `{artifacts_dir}/{module}/SPEC.md` using the template in `templates/SPEC.md`. The structure is:

#### Domain
- Entities (with key fields and invariants)
- Aggregates and ownership
- Multi-tenancy boundary (e.g., `company_id`)

#### Use cases
- One section per verb of the business: "Create sale", "Cancel appointment", "Refund commission"
- Each use case has: actor, preconditions, steps, postconditions, observable side effects

#### API surface
- Table of endpoints: method, path, auth, request, response, status codes
- Idempotency key behavior, if any
- Pagination/sorting defaults

#### Business rules
- Numbered list using `{conventions.business_rule_prefix}` (default `BR-01`, `BR-02`...)
- Each rule is **testable** — it asserts something observable
- Each rule cites its source: `(code: src/foo.ts:42)` or `(user-confirmed)` or `(legacy bug, intentional parity)`

#### Side effects
- Webhooks emitted (with event names and payload shape)
- Emails / notifications
- Queue messages
- Analytics events

#### Out of scope
- Explicit list of things this module does NOT do (or that we're explicitly not migrating)

#### Open questions
- Things you couldn't determine and need resolution before `/rdd-tests`

### 5. Hand-off

Tell the user:

- Path to `SPEC.md`
- Number of business rules captured
- Any open questions still pending
- Next step: `/rdd-tests {module}`

## Quality bar for business rules

A good business rule:

- ✅ "BR-04: Sale total must equal sum of line items minus discounts, rounded half-up to 2 decimals."
- ✅ "BR-09: A customer cannot have two appointments overlapping by ≥1 minute on the same employee's calendar."
- ✅ "BR-12: Stripe webhook with same event id must be idempotent (second delivery returns 200 without re-applying the side effect)."

A bad business rule (rewrite or drop):

- ❌ "BR-03: The system should be fast." (not testable)
- ❌ "BR-07: Use Postgres for storage." (implementation, not behavior)
- ❌ "BR-11: The code should be clean." (vague, not behavior)

## Anti-patterns to avoid

- **Don't describe implementation** — internal class structure, ORM choice, framework decorators. The spec is about what callers and the database see.
- **Don't smooth over bugs** — if the legacy returns 200 instead of 422 for invalid input, write that down. Migration locks current behavior; bug fixes are separate, marked changes.
- **Don't make rules up.** Every rule must trace to code, user statement, or explicit decision. Speculation goes to "Open questions".
- **Don't skip the user interview.** Code-only specs miss tribal knowledge and create regressions.

## When to spawn sub-agents

For modules with >20 entry points or >2000 lines of legacy code, use the Explore agent (very-thorough) to do the deep code read and return a structured summary. Then synthesize the spec yourself.
