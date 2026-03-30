---
name: rails-service-patterns
description: Analyzes and recommends Rails service object patterns for business logic extraction including command objects, result objects, form objects, query objects, and external API integration. Use when extracting logic from controllers/models, orchestrating multi-step workflows, or organizing app/services. NOT for simple CRUD, model validations, controller routing, or background job scheduling.
allowed-tools: Read, Grep, Glob
---

# Rails Service Object Patterns

Analyze and recommend patterns for extracting and organizing business logic in Rails applications.

## Quick Reference

| Pattern | Use When | Entry Point |
|---------|----------|-------------|
| Basic Service | Single operation with transaction | `CreateOrder.new(...).call` |
| Result Object | Caller needs success/failure + data | `Result.new(success?: true, data:)` |
| Form Object | Multi-model form submissions | `RegistrationForm.new(params).save` |
| Query Object | Complex reusable queries | `UserSearchQuery.new(scope).call` |
| Policy Object | Authorization decisions | `PostPolicy.new(user, post).update?` |

## Supporting Documentation

- [patterns.md](patterns.md) - Result objects, form objects, and profile-aware guidance

## Core Principles

1. **VerbNoun naming**: `CreateOrder`, `SendInvitation` -- never `OrderService` or `UserManager`
2. **One public method**: Expose only `call` (or `perform`)
3. **Explicit return values**: Use Result objects, never exceptions for expected flow control
4. **Profile-aware extraction**: See "When to Extract" below

## When to Extract a Service (Profile-Dependent)

| Scenario | Omakase | Service-Oriented / API-First |
|----------|---------|------------------------------|
| Logic on a single model's own data | Model method or concern | Model method |
| Shared behavior across models | Concern | Concern |
| Domain logic for one model | Concern | Service object |
| Multi-model workflow with rollback | Model method + transaction | Service object |
| External API call | Model method wrapping client | Service object |
| Simple side effect (email, log) | Callback (`after_commit`) | Service object |

**Omakase:** Only extract to a service when the workflow genuinely spans multiple unrelated models or external systems. Prefer concerns and enriched model methods.

**Service-oriented / API-first:** Service objects are the default extraction target for any non-trivial business logic.

## Result Object Pattern

Use `Struct.new(keyword_init: true)` for lightweight results. Never raise exceptions for expected failures (validation, auth, payment decline).

```ruby
class AuthenticateUser
  Result = Struct.new(:success?, :user, :error, keyword_init: true)

  def initialize(email:, password:)
    @email = email
    @password = password
  end

  def call
    user = User.find_by(email: @email)
    if user&.authenticate(@password)
      Result.new(success?: true, user: user)
    else
      Result.new(success?: false, error: "Invalid credentials")
    end
  end
end
```

See [patterns.md](patterns.md) for the enhanced monad-like `ServiceResult` with `on_success`/`on_failure` chaining.

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| God service (100+ lines) | Does too much | Split into composable services |
| Raising exceptions for flow control | Expensive, hard to handle | Use Result objects |
| Deep service-calls-service chains | Hidden coupling | Orchestrate from controller or coordinator |
| `self.call` class method pattern | No instance state, limits DI | Use instance methods with constructor DI |
| No return value | Caller can't react to failures | Always return Result or meaningful value |
| Service modifying passed-in objects | Surprising side effects | Return new objects or be explicit |
| VerbNoun naming violation (`UserService`) | Unclear responsibility, attracts god service | One service = one operation = one verb |

## Output Format

When analyzing or creating services, provide:
1. **Service file** in `app/services/` with VerbNoun naming
2. **Result struct** if callers need success/failure status
3. **Controller integration** showing how to call and handle results
4. **Test outline** covering happy path, failure cases, and edge cases
5. **Error handling** strategy (Result objects for expected failures, exceptions for unexpected)
