---
name: rails-controller-patterns
description: Analyzes and recommends Rails controller patterns including RESTful design, strong parameters, before_actions, response handling, and routing. Use when building controllers, defining routes, handling params, or managing request/response flow. NOT for model validations, service object internals, view templates, or background job logic.
allowed-tools: Read, Grep, Glob
---

# Rails Controller Patterns

Generate controllers following standard Rails RESTful conventions (7 actions, resourceful routing, before_actions, strong params).

## Supporting Documentation

- [patterns.md](patterns.md) - Detailed controller patterns and examples

## Core Principles

1. **Thin controllers**: Handle HTTP concerns only; delegate business logic. **Omakase:** delegate to models and concerns. **Service-oriented:** delegate to service objects
2. **RESTful by default**: Stick to 7 standard actions; create new resource controllers for custom actions
3. **Strong parameters always**: Never trust user input; use `params.expect` (Rails 8+)
4. **Consistent responses**: `redirect_to` after success, `render :action, status:` on failure
5. **One resource per controller**: Avoid multi-resource controllers
6. **Dedicated resource controllers**: Prefer `Cards::ClosuresController` over `post :close` on `CardsController`

## Strong Parameters: params.expect (Rails 8+)

`params.expect` replaces `params.require().permit()` with a cleaner, more secure API. Check `Gemfile.lock` for Rails version before using.

```ruby
# Basic
params.expect(post: [:title, :content])

# Arrays
params.expect(post: [:title, tags: []])

# Nested attributes
params.expect(user: [:name, :email, profile_attributes: [:bio, :avatar]])

# Dynamic hash attributes
params.expect(product: [:name, :price, metadata: {}])

# Legacy syntax (Rails < 8)
params.require(:post).permit(:title, :content)
```

## Dedicated Resource Controllers

Treat state changes as resources instead of adding custom member actions. This keeps controllers focused and RESTful.

```ruby
# Bad - custom actions bloat the controller
resources :cards do
  post :close
  post :reopen
end

# Good - resource controllers
resources :cards do
  scope module: :cards do
    resource :closure      # create = close, destroy = reopen
    resource :pin          # create = pin, destroy = unpin
  end
end
```

## Error Handling

```ruby
class ApplicationController < ActionController::Base
  rescue_from ActiveRecord::RecordNotFound, with: :not_found
  rescue_from ActionPolicy::Unauthorized, with: :forbidden

  private

  def not_found
    respond_to do |format|
      format.html { render "errors/not_found", status: :not_found }
      format.json { render json: { error: "Not found" }, status: :not_found }
    end
  end

  def forbidden
    respond_to do |format|
      format.html { redirect_back fallback_location: root_path, alert: "Not authorized." }
      format.json { render json: { error: "Forbidden" }, status: :forbidden }
    end
  end
end
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Fat controllers (50+ line actions) | Hard to test and maintain | **Omakase:** move logic to model methods/concerns. **Service-oriented:** extract to service objects |
| Business logic in controllers | Violates SRP | **Omakase:** move to models. **Service-oriented:** move to services |
| Custom member actions (`:close`, `:archive`) | Controller grows unbounded | Create dedicated resource controllers (`Cards::ClosuresController`) |
| `params.permit!` | Allows all params (mass assignment) | Use `params.expect` with explicit fields |
| Deeply nested routes (>1 level) | Confusing URLs and helpers | Use `shallow: true` or flat routes |
| Skipping authentication filters | Security vulnerability | Apply `before_action` broadly, skip selectively |
| Multiple `respond_to` blocks | Code duplication | Use `respond_to` once or separate API controllers |

## Status Code Checklist

- Return `status: :unprocessable_entity` (422) for validation failures
- Return `status: :not_found` (404) for missing records
- Return `status: :forbidden` (403) for authorization failures
- Return `status: :see_other` (303) for `redirect_to` after DELETE
- Use `rescue_from` in ApplicationController for consistent error responses

## Output Format

When analyzing or creating controllers, provide:
1. **Controller file** with proper structure and thin actions
2. **Route entries** for `config/routes.rb`
3. **Before actions** for auth, resource loading
4. **Strong params** method with explicit field list
5. **Error handling** strategy (rescue_from or inline)
