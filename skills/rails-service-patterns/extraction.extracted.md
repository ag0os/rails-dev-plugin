# Service Extraction — `extracted` axis

In an `extracted` project, service objects are the default home for non-trivial business logic. Controllers route, models validate, services orchestrate.

## When to extract

Extract to a service object for any non-trivial logic. Only trivial, single-model concerns stay off the service layer.

| Operation | Home |
|-----------|------|
| Trivial accessor / derived value on one model | Model method |
| Shared behavior across models | Concern |
| Domain logic for one model | Service object |
| Multi-model workflow with rollback | Service object wrapping an `ActiveRecord::Base.transaction` |
| External API call | Service object (wraps the client, owns retries/errors) |
| Simple side effect (email, log) | Service object — avoid callbacks for side effects |

## Conventions

- VerbNoun naming, one public `call`, Result object return (see SKILL.md and [patterns.md](patterns.md)).
- Place services in `app/services/`. Keep the directory flat until ~10 files, then group by domain (`app/services/orders/`, `app/services/payments/`).
- Side effects are explicit steps inside a service, not hidden in model lifecycle hooks. Reserve callbacks for data integrity only.

## What not to do

- Do not push workflow orchestration back into model callbacks "to save a file." That is the `native` axis; applying it here fights the codebase.
- Do not let services call services three deep — orchestrate from the controller or a single coordinator service.
