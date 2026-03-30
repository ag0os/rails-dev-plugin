---
name: rails-architecture-patterns
description: Provides architectural planning, design decisions, and coordination guidance for Rails applications. Use when planning new features, choosing between design approaches (STI vs polymorphic, service vs concern, monolith vs engine), evaluating system architecture, or deciding which domain skill or agent to delegate to. NOT for implementation details within a single domain (use the domain-specific skill instead).
allowed-tools: Read, Grep, Glob
---

# Rails Architecture Patterns

Architectural planning frameworks and decision guidance. Coordinate across domain skills and agents.

See [patterns.md](patterns.md) for detailed decision frameworks and profile-aware anti-pattern fixes.

## Stack Profile Awareness

**Before recommending patterns, determine the project's stack profile.** See `rails-stack-profiles` for detection logic.

| Decision | Omakase | Service-Oriented | API-First |
|----------|---------|-----------------|-----------|
| Business logic home | Models + concerns | Service objects | Service/command objects |
| Fat controller fix | Extract to concern/model | Extract to service | Extract to service |
| Testing | Minitest + fixtures | RSpec + FactoryBot | RSpec + FactoryBot |
| Auth | `has_secure_password` | Devise | JWT/token |
| Job backend | Solid Queue | Sidekiq | Sidekiq or Solid Queue |

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
| Shared logic | Concern | Service Object | Omakase, or adds model capability | Service-oriented, or orchestrates workflow |
| Background work | ActiveJob | Inline | > 100ms or external call | Fast + must be synchronous |
| Job backend | Solid Queue | Sidekiq | Omakase, simple needs | Redis already in stack |
| API format | JSON:API | Custom JSON | Public API, many clients | Internal API, simple needs |
| Frontend | Hotwire | SPA (React) | Content-heavy, progressive | Complex interactive UI |
| Auth | `has_secure_password` | Devise | Omakase, simple needs | Multi-feature auth needed |

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

**Universal (all profiles):**
- [ ] No business logic in controllers or views
- [ ] Controllers have 7 or fewer actions
- [ ] No N+1 queries (check with Bullet gem)
- [ ] Background jobs are idempotent
- [ ] Test coverage on critical paths
- [ ] No circular dependencies between models

**Omakase:**
- [ ] Models decomposed via concerns (not monolithic god models)
- [ ] Callbacks are simple and predictable (data integrity, not workflows)
- [ ] Using Rails defaults (Solid Queue, etc.) where possible

**Service-oriented:**
- [ ] Models under 200 lines (extract services if larger)
- [ ] All external API calls go through service objects
- [ ] Service objects have single responsibility
- [ ] Result pattern used consistently

**API-first:**
- [ ] All responses use serializers (never raw ActiveRecord)
- [ ] API versioned from day one
- [ ] Authentication is token-based
- [ ] Error responses follow consistent envelope format

## Output Format

```
## Architecture Plan: [feature_name]

**Detected Profile:** [omakase | service-oriented | api-first]
**Layers Involved:** Models, Controllers, Services, Tests

**Implementation Steps:**
1. [step] -- delegate to [skill/agent]
2. ...

**Design Decisions:**
- [decision]: [rationale]

**Risks:**
- [risk]: [mitigation]
```
