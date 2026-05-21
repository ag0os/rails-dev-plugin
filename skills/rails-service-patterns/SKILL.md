---
name: rails-service-patterns
description: Analyzes and recommends Rails service object patterns for business logic extraction including command objects, result objects, form objects, query objects, and external API integration. Use when extracting logic from controllers/models, orchestrating multi-step workflows, or organizing app/services. NOT for simple CRUD, model validations, controller routing, or background job scheduling.
allowed-tools: Read, Grep, Glob
---

# Rails Service Object Patterns

Analyze and recommend patterns for extracting and organizing business logic in Rails applications.

## Resolve the axis first

This skill forks on **Axis A — logic placement**. Before recommending where logic goes:

1. If the axes are already resolved this session, use them.
2. Otherwise resolve via `rails-stack-profiles` (Axis A only — `native` or `extracted`).
3. Then read the matching extraction guide:
   - `native` → [extraction.native.md](extraction.native.md)
   - `extracted` → [extraction.extracted.md](extraction.extracted.md)

If the axis cannot be resolved, default to `native` and state the assumption.

Everything else in this file is **invariant** — it holds regardless of axis.

## Quick Reference

| Pattern | Use When | Entry Point |
|---------|----------|-------------|
| Basic Service | Single operation with transaction | `CreateOrder.new(...).call` |
| Result Object | Caller needs success/failure + data | `Result.new(success?: true, data:)` |
| Form Object | Multi-model form submissions | `RegistrationForm.new(params).save` |
| Query Object | Complex reusable queries | `UserSearchQuery.new(scope).call` |
| Policy Object | Authorization decisions | `PostPolicy.new(user, post).update?` |

## Supporting Documentation

- [extraction.native.md](extraction.native.md) — when to extract, `native` axis
- [extraction.extracted.md](extraction.extracted.md) — when to extract, `extracted` axis
- [patterns.md](patterns.md) — result objects, form objects, query objects, error handling

## Core Principles

1. **VerbNoun naming**: `CreateOrder`, `SendInvitation` -- never `OrderService` or `UserManager`
2. **One public method**: Expose only `call` (or `perform`)
3. **Explicit return values**: Use Result objects, never exceptions for expected flow control
4. **Axis-aware extraction**: Whether a given operation becomes a service at all depends on Axis A. Read the matching extraction guide above before recommending a service.

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
1. **Service file** with VerbNoun naming, placed per the axis extraction guide
2. **Result struct** if callers need success/failure status
3. **Controller integration** showing how to call and handle results
4. **Test outline** covering happy path, failure cases, and edge cases
5. **Error handling** strategy (Result objects for expected failures, exceptions for unexpected)
