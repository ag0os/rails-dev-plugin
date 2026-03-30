---
name: rails-api-patterns
description: Analyzes Rails API controllers, serializers, and endpoint design for best practices. Use when building RESTful API endpoints, implementing JWT authentication, designing JSON response structures, adding API versioning, writing serializers, setting up error handling, or adding pagination and rate limiting. NOT for HTML views, Turbo/Stimulus interactions, or GraphQL schemas.
allowed-tools: Read, Grep, Glob
---

# Rails API Patterns

Analyze and recommend best practices for Rails API design including controllers, serialization, authentication, versioning, and error handling.

Follow standard Rails API controller conventions (`ActionController::API`, strong parameters, `rescue_from`). See [patterns.md](patterns.md) for opinionated decisions and configuration.

## Quick Reference

| Area | Decision | Key File |
|------|----------|----------|
| Base controller | `ActionController::API` + shared concerns | `app/controllers/api/base_controller.rb` |
| Versioning | URL namespace (`/api/v1/`) -- NOT header-based | `config/routes.rb` |
| Auth | JWT bearer token via `JwtService` | `app/controllers/concerns/authenticatable.rb` |
| Serialization | Blueprinter preferred (views for complexity) | `app/blueprints/` |
| Response format | `{ data:, meta:, errors: }` envelope always | All API responses |
| Pagination | Pagy or Kaminari + `max_per_page: 100` cap | `app/controllers/concerns/paginatable.rb` |
| Rate limiting | Rack::Attack: 100/min per token, 20/min per IP | `config/initializers/rack_attack.rb` |
| CORS | `rack-cors` gem, env-driven origins | `config/initializers/cors.rb` |

## Core Decisions

1. **Response envelope always** -- every response wraps in `{ data:, meta:, errors: }`, never bare objects/arrays
2. **URL versioning from day one** -- `/api/v1/` namespace even for first version; header-based versioning adds complexity without benefit for most apps
3. **Blueprinter over AMS** -- ActiveModel::Serializer is unmaintained; prefer Blueprinter for views-based serialization, or Jbuilder for complex one-offs
4. **Cap per_page at 100** -- enforce `[params[:per_page].to_i, 100].min` to prevent clients from requesting unbounded results
5. **Authenticate in base, skip explicitly** -- `before_action :authenticate` in `Api::BaseController`, use `skip_before_action` for public endpoints
6. **Dual rate limits** -- per-token (100/min) for authenticated, per-IP (20/min) for unauthenticated; return `Retry-After` header

## Anti-Patterns

| Bad | Good | Why |
|-----|------|-----|
| Render ActiveRecord objects directly | Use serializers/blueprints | Controls exposed fields, prevents leaking columns |
| No pagination on `index` | Always paginate with max cap | Memory + performance, prevents client abuse |
| Version in header only | URL versioning (`/api/v1/`) | Simpler routing, cacheable, easier debugging |
| N+1 queries in serializers | `includes()` in controller | Serializers should not trigger queries |
| Bare error strings | Structured `{ error:, errors: }` envelope | Clients need machine-parseable error format |
| CORS `origins "*"` in production | Env-driven origin allowlist | Security; wildcard disables credential sharing |

## Output Format

When reporting on API design, use:

```
## API Analysis: [controller_or_endpoint]

**Issues:**
- [severity] description

**Recommendations:**
1. actionable recommendation
```
