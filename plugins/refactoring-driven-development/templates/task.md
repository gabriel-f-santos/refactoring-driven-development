# {NN} — {entry-point-name}

> One unit of port work — corresponds to one legacy entry point (HTTP route, queue consumer, scheduled job, webhook, etc.). Created by `/rdd-refactor-04` when the port phase reaches this entry point. Aggregated state lives in `../PROGRESS.md`.

**Status:** pending | in_progress | completed | skipped | failed

## Source

- **Legacy file(s):** `<path/to/legacy/handler.ts>`
- **Target file(s):** `<path/to/new/controller.ts>`

## Spec slice

> Subset of `../spec/SPEC.md` that this entry point implements.

- **API surface:** `<METHOD /path>` — auth: `<scheme>`
- **BRs covered:** BR-NN, BR-NN, ...
- **Error codes emitted:** `<CODE_1>`, `<CODE_2>`, ...
- **Side effects triggered:** `<list or "none">`

## Tests

- **Tests on new (`TEST_TARGET=new`):** X/Y passing
- **Tests on legacy (`TEST_TARGET=legacy`):** X/Y passing — parity ✅ | drift ⚠
- **Test files:** `<path/to/spec.ts>`

## Fix-loop attempts (if any)

- **Attempt 1:** <one-line summary, root-cause hypothesis, fix>
- **Attempt 2:** ...
- **Attempt 3:** ...

> If 3 attempts exhausted on the same test, the skill stops and surfaces this section.

## Observations

> Out-of-scope findings recorded during port. Don't act on them — record only. Examples: "agent-relatorios uses query-intelligence helper which lives in task 03 — wait for that task to finalize the import."

- (none)

## Timestamps

- **Started:** YYYY-MM-DD HH:MM
- **Completed:** YYYY-MM-DD HH:MM
