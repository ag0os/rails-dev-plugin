---
name: rails-stack-profiles
description: Detects and applies Rails stack profiles (omakase, service-oriented, api-first) based on project conventions. Use when planning architecture, making design decisions, or when the recommended approach depends on the project's stack choices (Solid Queue vs Sidekiq, concerns vs service objects, Minitest vs RSpec, fixtures vs factories). NOT for implementation details — delegates to domain-specific skills.
allowed-tools: Read, Grep, Glob
---

# Rails Stack Profiles

Detect the project's architectural profile before recommending patterns. Different Rails codebases follow different conventions — recommendations must match the project, not a single opinionated default.

See [profiles.md](profiles.md) for detailed profile definitions, detection signals, and per-profile recommendations.

## Three Profiles

| Profile | Philosophy | Key Markers |
|---------|-----------|-------------|
| **omakase** | Rails defaults, convention over gems | Solid Queue, Minitest, fixtures, concerns, `has_secure_password` |
| **service-oriented** | Explicit layers, extracted business logic | Sidekiq, RSpec, FactoryBot, `app/services/`, Devise, Pundit |
| **api-first** | Headless JSON backend | `ActionController::API`, serializers, JWT, no `app/views/` |

## Detection Checklist

Run these checks against the project to determine its profile:

```
1. Read Gemfile for key gems
2. Check directory structure (app/services/, spec/ vs test/, app/views/)
3. Check config/database.yml adapter
4. Check test setup (RSpec vs Minitest, factories vs fixtures)
5. Check job backend (config/queue.yml vs config/sidekiq.yml)
6. Check auth approach (Devise, has_secure_password, JWT)
```

## Quick Detection Matrix

| Signal | Omakase | Service-Oriented | API-First |
|--------|---------|-----------------|-----------|
| `gem "sidekiq"` | | X | X |
| `gem "solid_queue"` | X | | |
| `gem "rspec-rails"` | | X | X |
| `test/` directory | X | | |
| `gem "factory_bot"` | | X | X |
| `test/fixtures/` | X | | |
| `app/services/` exists | | X | |
| `gem "devise"` | | X | |
| `has_secure_password` | X | | |
| `gem "pundit"` | | X | |
| `gem "jbuilder"` | X | | |
| `gem "alba"` or `gem "blueprinter"` | | X | X |
| `ActionController::API` base | | | X |
| `app/views/` has ERB files | X | X | |
| `gem "jwt"` | | | X |
| `config/solid_cache.yml` | X | | |

## Profile Mixing

Most real projects are **hybrids**. A project can be:
- Omakase core with a service-oriented billing module
- Service-oriented with an api-first namespace (`/api/v1/`)
- Omakase that adopted RSpec early but kept everything else default

**When signals conflict:** weight the project's dominant pattern. A project with `app/services/` containing 50 classes is service-oriented even if it uses Minitest.

## How Profiles Affect Recommendations

| Decision Point | Omakase | Service-Oriented | API-First |
|---------------|---------|-----------------|-----------|
| Where does business logic go? | Model methods + concerns | Service objects | Service objects or interactors |
| Fat controller fix | Extract to model/concern | Extract to service object | Extract to service object |
| God model fix | Extract concerns | Extract service objects + value objects | Extract query/command objects |
| Callbacks for side effects? | Acceptable if simple | Avoid — use services | Avoid — use services |
| Testing framework | Minitest | RSpec | RSpec |
| Test data | Fixtures | FactoryBot | FactoryBot |
| Job backend | Solid Queue | Sidekiq | Sidekiq or Solid Queue |
| Auth | `has_secure_password` | Devise | JWT or token-based |
| Authorization | Controller-level checks | Pundit policies | Token scopes or Pundit |
| Frontend | Hotwire (Turbo + Stimulus) | Hotwire (Turbo + Stimulus) | None — JSON responses |
| Serialization | Jbuilder or `to_json` | Alba, Blueprinter | Alba, Blueprinter, jsonapi-serializer |
| Real-time | Solid Cable + Turbo Streams | ActionCable + Turbo Streams | WebSockets or SSE |
| Cache backend | Solid Cache | Redis | Redis |
| DB default | SQLite (dev), PostgreSQL (prod) | PostgreSQL | PostgreSQL |

## Output Format

When reporting a detected profile:

```
## Stack Profile: [omakase | service-oriented | api-first | hybrid]

**Detected from:**
- [signal]: [evidence]
- [signal]: [evidence]

**Hybrid notes:** [if applicable — which parts diverge and why]

**Recommendations will follow [profile] conventions.**
```
