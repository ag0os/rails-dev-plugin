---
name: rails-api-patterns
description: Analyzes Rails API controllers, serializers, and endpoint design for best practices. Use when building RESTful API endpoints, implementing JWT authentication, designing JSON response structures, adding API versioning, writing serializers, setting up error handling, or adding pagination and rate limiting. NOT for HTML views, Turbo/Stimulus interactions, or GraphQL schemas.
allowed-tools: Read, Grep, Glob
---

# Rails API Patterns

Analyze and recommend best practices for Rails API design including controllers, serialization, authentication, versioning, and error handling.

See [patterns.md](patterns.md) for full code examples.

## Quick Reference

| Area | Pattern | Key File |
|------|---------|----------|
| Base controller | `ActionController::API` + shared concerns | `app/controllers/api/base_controller.rb` |
| Versioning | URL namespace (`/api/v1/`) | `config/routes.rb` |
| Auth | JWT bearer token | `app/controllers/concerns/authenticatable.rb` |
| Serialization | ActiveModel::Serializer or Blueprinter | `app/serializers/` |
| Errors | Consistent JSON envelope | `rescue_from` in base controller |
| Pagination | Kaminari or Pagy + meta object | Controller `index` actions |
| Rate limiting | Rack::Attack per token/IP | `config/initializers/rack_attack.rb` |

## Core Principles

1. **Consistent response envelope** -- always wrap in `{ data:, meta:, errors: }`
2. **Proper HTTP status codes** -- 200, 201, 204, 400, 401, 403, 404, 422, 429, 500
3. **Versioned from day one** -- `/api/v1/` namespace even for first version
4. **Authenticate every request** -- `before_action :authenticate` in base controller
5. **Paginate all collections** -- never return unbounded lists
6. **Prevent N+1 queries** -- eager load associations in controller

## Key Patterns

### Base API Controller

```ruby
class Api::BaseController < ActionController::API
  include ActionController::HttpAuthentication::Token::ControllerMethods

  before_action :authenticate

  rescue_from ActiveRecord::RecordNotFound,    with: :not_found
  rescue_from ActiveRecord::RecordInvalid,     with: :unprocessable_entity
  rescue_from ActionController::ParameterMissing, with: :bad_request

  private

  def authenticate
    authenticate_or_request_with_http_token do |token, _options|
      @current_user = User.find_by(api_token: token)
    end
  end

  def not_found(exception)
    render json: { error: exception.message }, status: :not_found
  end

  def unprocessable_entity(exception)
    render json: { errors: exception.record.errors }, status: :unprocessable_entity
  end

  def bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end
end
```

### RESTful Resource Controller

```ruby
class Api::V1::ProductsController < Api::BaseController
  def index
    products = Product.includes(:category).page(params[:page]).per(params[:per_page])
    render json: { data: products, meta: pagination_meta(products) }
  end

  def show
    render json: { data: Product.find(params[:id]) }
  end

  def create
    product = Product.new(product_params)
    if product.save
      render json: { data: product }, status: :created
    else
      render json: { errors: product.errors }, status: :unprocessable_entity
    end
  end

  private

  def product_params
    params.expect(product: [:name, :price, :description, :category_id])
  end
end
```

### JSON Response Envelope

```json
{
  "data": { "id": "123", "type": "products", "attributes": { "name": "Widget" } },
  "meta": { "total": 100, "page": 1, "per_page": 20 }
}
```

### Error Response

```json
{
  "error": "Record not found",
  "errors": { "name": ["can't be blank"] }
}
```

## Anti-Patterns

| Bad | Good | Why |
|-----|------|-----|
| Render ActiveRecord objects directly | Use serializers | Controls exposed fields |
| No pagination on `index` | Always paginate | Memory + performance |
| Generic 500 for all errors | Specific status codes | Client error handling |
| Auth logic in every controller | Shared concern / base class | DRY |
| Version in header only | URL versioning (`/api/v1/`) | Simpler routing, caching |
| N+1 queries in serializers | `includes()` in controller | Performance |

## Versioning Strategy

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :products, only: [:index, :show, :create, :update, :destroy]
    resource :auth, only: [:create], controller: "auth"
  end
end
```

## Output Format

When reporting on API design, use:

```
## API Analysis: [controller_or_endpoint]

**Issues:**
- [severity] description

**Recommendations:**
1. actionable recommendation
```
