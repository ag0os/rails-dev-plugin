# Service Extraction — `native` axis

In a `native` project, business logic lives in models, concerns, and plain Ruby objects. A service object is the exception, not the default.

## When to extract

Default to keeping logic on the model. Reach for a separate object **only** when a workflow genuinely spans multiple unrelated models or an external system — and even then, prefer a PORO over a `Service`-suffixed class.

| Operation | Home |
|-----------|------|
| Logic on a single model's own data | Model method |
| Shared behavior across models | Concern |
| Domain logic for one model | Concern (slice by trait: `Triageable`, `Closeable`) |
| Multi-model workflow with rollback | Model method wrapping an `ActiveRecord::Base.transaction` |
| External API call | Model method wrapping a thin client object |
| Simple side effect (email, log) | `after_commit` callback |

## When you do extract

A workflow spanning several unrelated models or systems earns its own object. Keep it a PORO:

- Name it for the operation (`OrderFulfillment`, not `OrderService`).
- Place it in `app/models/` or `app/operations/` — a dedicated `app/services/` directory is not the `native` convention.
- The Result object pattern (see [patterns.md](patterns.md)) still applies for callers that branch on outcome.

## What not to do

- Do not create `app/services/` and route every multi-step method through it. That is the `extracted` axis; applying it here fights the codebase.
- Do not replace a working `after_commit` side effect with a service "for consistency."
