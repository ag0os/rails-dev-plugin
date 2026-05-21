---
name: rails-architecture-patterns
description: Provides architectural planning, design decisions, and coordination guidance for Rails applications. Use when planning new features, choosing between design approaches (STI vs polymorphic, service vs concern, monolith vs engine), evaluating system architecture, or deciding which domain skill or agent to delegate to. NOT for implementation details within a single domain (use the domain-specific skill instead).
allowed-tools: Read, Grep, Glob
---

# Rails Architecture Patterns

Architectural planning frameworks and decision guidance. Coordinate across domain skills and agents.

See [patterns.md](patterns.md) for detailed decision frameworks. Logic-placement and structural-smell fixes fork on Axis A — see [decisions.native.md](decisions.native.md) / [decisions.extracted.md](decisions.extracted.md).

## Resolve the Stack Axes First

**Before recommending patterns, resolve the project's architecture axes** via `rails-stack-profiles` (once per session):

- **Axis A — logic placement** (`native` / `extracted`): drives where business logic lives and how structural smells are fixed. See [decisions.native.md](decisions.native.md) / [decisions.extracted.md](decisions.extracted.md).
- **Axis B — delivery** (`html` / `api`): drives view vs serializer choices and which domain skills apply.

Orthogonal facts — test framework, job backend, cache store, auth library — come from the `project-conventions` fingerprint, not from an architecture axis. Never infer them here.

## Available Domain Skills and Agents

| Domain | Skill | Agent | Covers |
|--------|-------|-------|--------|
| Models | rails-model-patterns | rails-model | ActiveRecord, associations, migrations, scopes |
| Controllers | rails-controller-patterns | rails-controller | RESTful actions, params, routes, error handling |
| API | rails-api-patterns | rails-api | API controllers, serialization, versioning, JWT |
| Services | rails-service-patterns | rails-service | Service objects, result pattern, DI, external APIs |
| Jobs | rails-jobs-patterns | rails-jobs | ActiveJob, Sidekiq, idempotency, batch processing |
| GraphQL | rails-graphql-patterns | rails-graphql | Schema, mutations, DataLoader, subscriptions |
| Hotwire | hotwire-patterns | rails-hotwire | Stimulus controllers, Turbo frames/streams |
| Views | rails-views-patterns | rails-views | ERB, partials, helpers, caching, accessibility |
| Testing | rails-testing-patterns | rails-test | RSpec, Minitest, system/integration tests |
| DevOps | rails-devops-patterns | rails-devops | Docker, CI/CD, monitoring, security |
| Mailers | rails-mailer-patterns | (skill only) | Action Mailer, previews, delivery, interceptors |
| Auth | rails-auth-patterns | (skill only) | has_secure_password, Devise, sessions, password reset |
| Caching | rails-caching-patterns | (skill only) | Fragment, low-level, HTTP caching, invalidation |
| Refactoring | ruby-refactoring | (skill only) | Code smells, refactoring patterns |
| Object Design | ruby-object-design | (skill only) | Class vs module, Struct, Data, design decisions |

## Quick Decision Matrix

| Question | Option A | Option B | Choose A when | Choose B when |
|----------|----------|----------|---------------|---------------|
| Inheritance | STI | Polymorphic | >70% shared columns, same behavior | Different attributes per type |
| Shared logic | Concern | Service Object | Axis A `native`, or adds a model capability | Axis A `extracted`, or orchestrates a workflow |
| Background work | ActiveJob | Inline | > 100ms or external call | Fast + must be synchronous |
| Job backend | Solid Queue | Sidekiq | Simple needs, no Redis in stack | Redis already in stack |
| API format | JSON:API | Custom JSON | Public API, many clients | Internal API, simple needs |
| Frontend | Hotwire | SPA (React) | Content-heavy, progressive | Complex interactive UI |
| Auth | `has_secure_password` | Devise | Simple auth needs | Multi-feature auth needed |

## Implementation Order

For a typical feature:

```
1. Migration + Model     (rails-model)
2. Routes + Controller   (rails-controller or rails-api)
3. Views or Serializer   (rails-views or rails-api)
4. Service objects        (rails-service) -- if complex logic
5. Background jobs        (rails-jobs) -- if async work needed
6. Tests                  (rails-testing-patterns)
```

## Architecture Health Checks

**Universal (all projects):**
- [ ] No business logic in controllers or views
- [ ] Controllers have 7 or fewer actions
- [ ] No N+1 queries (check with Bullet gem)
- [ ] Background jobs are idempotent
- [ ] Test coverage on critical paths
- [ ] No circular dependencies between models

**Axis A — `native`:**
- [ ] Models decomposed via concerns (not monolithic god models)
- [ ] Callbacks are simple and predictable (data integrity, not workflows)
- [ ] Using Rails defaults where possible

**Axis A — `extracted`:**
- [ ] Models stay lean — non-trivial logic extracted to services
- [ ] All external API calls go through service objects
- [ ] Service objects have single responsibility
- [ ] Result pattern used consistently

**Axis B — `api`:**
- [ ] All responses use serializers (never raw ActiveRecord)
- [ ] API versioned from day one
- [ ] Authentication is token-based
- [ ] Error responses follow consistent envelope format

## Output Format

```
## Architecture Plan: [feature_name]

**Stack Axes:** logic=[native | extracted], delivery=[html | api]
**Layers Involved:** Models, Controllers, Services, Tests

**Implementation Steps:**
1. [step] -- delegate to [skill/agent]
2. ...

**Design Decisions:**
- [decision]: [rationale]

**Risks:**
- [risk]: [mitigation]
```
