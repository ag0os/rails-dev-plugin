# Rails API Patterns -- Detailed Reference

Follow standard Rails conventions for API controllers, JWT encode/decode, serializer basics, and RESTful actions. This file covers only the opinionated decisions and non-obvious configuration.

## Response Envelope Convention

Every API response MUST use this structure -- no bare objects or arrays:

```json
{
  "data": { ... },
  "meta": { "current_page": 1, "total_pages": 3, "total_count": 42, "per_page": 20 },
  "errors": null
}
```

Error responses:

```json
{
  "data": null,
  "error": "Record not found",
  "errors": { "name": ["can't be blank"], "price": ["must be greater than 0"] }
}
```

Use `error` (singular) for single-message errors (404, 401, 429). Use `errors` (plural hash) for validation errors (422).

## Pagination Concern

The key opinions: default 20 per page, hard cap at 100, include `next_page`/`prev_page` in meta.

```ruby
# app/controllers/concerns/paginatable.rb
module Paginatable
  extend ActiveSupport::Concern

  DEFAULTS = { page: 1, per_page: 20, max_per_page: 100 }.freeze

  private

  def paginate(scope)
    pp = page_params
    scope.page(pp[:page]).per(pp[:per_page])
  end

  def page_params
    page     = (params[:page] || DEFAULTS[:page]).to_i
    per_page = [(params[:per_page] || DEFAULTS[:per_page]).to_i, DEFAULTS[:max_per_page]].min
    { page: page, per_page: per_page }
  end

  def pagination_meta(collection)
    {
      current_page: collection.current_page,
      next_page: collection.next_page,
      prev_page: collection.prev_page,
      total_pages: collection.total_pages,
      total_count: collection.total_count
    }
  end
end
```

## Rate Limiting Configuration

Dual-strategy: per-token for authenticated requests, per-IP for unauthenticated. Always return rate limit headers.

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Authenticated: 100 requests/minute keyed by token
  throttle("api/token", limit: 100, period: 1.minute) do |req|
    if req.path.start_with?("/api/")
      req.env["HTTP_AUTHORIZATION"]&.split("Bearer ")&.last
    end
  end

  # Unauthenticated: 20 requests/minute keyed by IP
  throttle("api/ip", limit: 20, period: 1.minute) do |req|
    req.ip if req.path.start_with?("/api/") && req.env["HTTP_AUTHORIZATION"].blank?
  end

  self.throttled_responder = lambda do |matched, env|
    headers = {
      "Content-Type" => "application/json",
      "Retry-After" => (Time.current + matched[:period]).httpdate,
      "X-RateLimit-Limit" => matched[:limit].to_s,
      "X-RateLimit-Remaining" => "0"
    }
    body = { error: "Rate limit exceeded. Retry after #{matched[:period]} seconds." }.to_json
    [429, headers, [body]]
  end
end
```

## CORS Configuration

Env-driven origin allowlist. Expose rate limit headers. Never use `origins "*"` with credentials.

```ruby
# config/initializers/cors.rb
Rails.application.config.middleware.insert_before 0, Rack::Cors do
  allow do
    origins ENV.fetch("ALLOWED_ORIGINS", "http://localhost:3000").split(",")
    resource "/api/*",
      headers: :any,
      methods: [:get, :post, :put, :patch, :delete, :options],
      expose: ["X-RateLimit-Limit", "X-RateLimit-Remaining", "Retry-After"],
      max_age: 600
  end
end
```

## Versioning Strategy

Use URL namespacing. When introducing v2, keep v1 controllers intact and create new v2 controllers with updated serializers. Share models across versions.

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :products
    resource :auth, only: [:create], controller: "auth"
  end

  namespace :v2 do
    resources :products  # New blueprint, same model
  end
end
```

Key rule: never mutate v1 response shapes after clients depend on them. Version the serializer/blueprint, not the model.

## Blueprinter View Pattern

Use Blueprinter views to control response detail level per endpoint:

```ruby
class ProductBlueprint < Blueprinter::Base
  identifier :id
  fields :name, :description, :created_at

  field :price do |product|
    "$#{'%.2f' % product.price}"
  end

  view :extended do
    association :category, blueprint: CategoryBlueprint
    association :reviews, blueprint: ReviewBlueprint
  end
end

# Controller: render with view selection
render json: { data: ProductBlueprint.render_as_hash(product, view: :extended) }
```

Prefer `:extended` view for `show` and default (no view) for `index` to minimize payload on collection endpoints.

## Error Status Code Conventions

| Status | When to use |
|--------|-------------|
| 409 Conflict | Stale update (optimistic locking via `lock_version`) |
| 403 vs 401 | 401 = no/invalid token; 403 = valid token but lacks permission |
| 204 vs 200 | 204 for DELETE (no body); 200 for soft-delete if returning updated record |
| 422 vs 400 | 422 = validation errors on a well-formed request; 400 = malformed/missing params |
