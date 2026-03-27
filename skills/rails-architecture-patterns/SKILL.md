---
name: rails-architecture-patterns
description: Provides architectural planning, design decisions, and coordination guidance for Rails applications. Use when planning new features, choosing between design approaches (STI vs polymorphic, service vs concern, monolith vs engine), evaluating system architecture, or deciding which domain skill or agent to delegate to. NOT for implementation details within a single domain (use the domain-specific skill instead).
allowed-tools: Read, Grep, Glob
---

# Rails Architecture Patterns

Provide architectural planning frameworks, design principles, and decision guidance for Rails applications. Coordinate across domain skills and agents.

See [patterns.md](patterns.md) for detailed decision frameworks and anti-patterns.

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
| Refactoring | ruby-refactoring | (skill only) | Code smells, refactoring patterns, Ruby Science |
| Object Design | ruby-object-design | (skill only) | Class vs module, Struct, Data, design decisions |

## Stack Profile Awareness

**Before recommending patterns, determine the project's stack profile.** See `rails-stack-profiles` for detection logic. The profile determines which patterns are appropriate:

| Decision | Omakase | Service-Oriented | API-First |
|----------|---------|-----------------|-----------|
| Business logic home | Models + concerns | Service objects | Service/command objects |
| Fat controller fix | Extract to concern/model | Extract to service | Extract to service |
| Testing | Minitest + fixtures | RSpec + FactoryBot | RSpec + FactoryBot |
| Auth | `has_secure_password` | Devise | JWT/token |
| Job backend | Solid Queue | Sidekiq | Sidekiq or Solid Queue |

## Core Design Principles

| Principle | Meaning | Rails Example |
|-----------|---------|---------------|
| **CoC** | Convention over Configuration | Follow Rails naming, directory structure |
| **DRY** | Don't Repeat Yourself | Extract concerns, shared partials, service objects |
| **RESTful** | Resource-oriented routes | 7 standard actions per controller |
| **Fat Model, Skinny Controller** | Business logic in models (omakase) or services (service-oriented) | Controllers only route + render |
| **Least Surprise** | Code does what reader expects | Follow the project's own conventions |
| **YAGNI** | Don't build what you don't need | Avoid premature abstraction |

## Planning Framework

When receiving a feature request:

1. **Analyze scope** -- which layers of the stack are involved?
2. **Identify models** -- what data entities and relationships are needed?
3. **Plan order** -- Models -> Controllers -> Views/Services -> Tests
4. **Choose patterns** -- service object? concern? background job?
5. **Delegate** -- route to the appropriate domain skill/agent
6. **Validate** -- review integration points between layers

## Quick Decision Matrix

| Question | Option A | Option B | Choose A when | Choose B when |
|----------|----------|----------|---------------|---------------|
| Inheritance | STI | Polymorphic | Shared behavior + same columns | Different attributes per type |
| Shared logic | Concern | Service Object | Omakase profile, or adds model capability | Service-oriented profile, or orchestrates a workflow |
| Background work | ActiveJob | Inline | > 100ms or external call | Fast + must be synchronous |
| Job backend | Solid Queue | Sidekiq | Omakase profile, simple needs | Service-oriented/api-first, Redis already in stack |
| API format | JSON:API | Custom JSON | Public API, many clients | Internal API, simple needs |
| Frontend | Hotwire | SPA (React) | Content-heavy, progressive | Complex interactive UI |
| Auth | `has_secure_password` | Devise | Omakase profile, simple needs | Service-oriented, standard multi-feature auth |

## Implementation Order

For a typical feature (e.g., "add product reviews"):

```
1. Migration + Model     (rails-model)
2. Routes + Controller   (rails-controller or rails-api)
3. Views or Serializer   (rails-views or rails-api)
4. Service objects        (rails-service) -- if complex logic
5. Background jobs        (rails-jobs) -- if async work needed
6. Tests                  (rails-testing-patterns)
```

## Architecture Health Checks

When reviewing an existing codebase, check (adapt to detected profile):

**Universal (all profiles):**
- [ ] No business logic in controllers or views
- [ ] Controllers have 7 or fewer actions
- [ ] No N+1 queries (check with Bullet gem)
- [ ] Background jobs are idempotent
- [ ] Test coverage on critical paths
- [ ] No circular dependencies between models

**Omakase profile:**
- [ ] Models decomposed via concerns (not monolithic god models)
- [ ] Callbacks are simple and predictable (data integrity, not workflows)
- [ ] Using Rails defaults (Solid Queue, etc.) where possible

**Service-oriented profile:**
- [ ] Models under 200 lines (extract services if larger)
- [ ] All external API calls go through service objects
- [ ] Service objects have single responsibility
- [ ] Result pattern used consistently

**API-first profile:**
- [ ] All responses use serializers (never raw ActiveRecord)
- [ ] API versioned from day one
- [ ] Authentication is token-based
- [ ] Error responses follow consistent envelope format

## Output Format

When providing architectural guidance, use:

```
## Architecture Plan: [feature_name]

**Layers Involved:** Models, Controllers, Services, Tests

**Implementation Steps:**
1. [step] -- delegate to [skill/agent]
2. ...

**Design Decisions:**
- [decision]: [rationale]

**Risks:**
- [risk]: [mitigation]
```
