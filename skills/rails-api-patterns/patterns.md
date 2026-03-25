# Rails API Patterns -- Detailed Reference

## Base Controller with Full Error Handling

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  include ActionController::HttpAuthentication::Token::ControllerMethods

  before_action :authenticate
  after_action :set_rate_limit_headers

  rescue_from ActiveRecord::RecordNotFound,       with: :not_found
  rescue_from ActiveRecord::RecordInvalid,         with: :unprocessable_entity
  rescue_from ActionController::ParameterMissing,  with: :bad_request
  rescue_from JWT::DecodeError,                    with: :unauthorized

  private

  def authenticate
    token = extract_token
    return render_unauthorized unless token

    payload = JwtService.decode(token)
    @current_user = User.find(payload[:user_id])
  rescue ActiveRecord::RecordNotFound
    render_unauthorized
  end

  def extract_token
    request.headers["Authorization"]&.split("Bearer ")&.last
  end

  def render_unauthorized
    render json: { error: "Invalid or missing authentication token" }, status: :unauthorized
  end

  def not_found(exception)
    render json: { error: exception.message }, status: :not_found
  end

  def unprocessable_entity(exception)
    render json: { errors: exception.record.errors.as_json }, status: :unprocessable_entity
  end

  def bad_request(exception)
    render json: { error: exception.message }, status: :bad_request
  end

  def pagination_meta(collection)
    {
      current_page: collection.current_page,
      total_pages: collection.total_pages,
      total_count: collection.total_count,
      per_page: collection.limit_value
    }
  end

  def set_rate_limit_headers
    # Set by Rack::Attack middleware, forwarded to response
  end
end
```

## JWT Authentication Service

```ruby
# app/services/jwt_service.rb
class JwtService
  SECRET = Rails.application.secret_key_base
  ALGORITHM = "HS256"
  DEFAULT_EXPIRY = 24.hours

  def self.encode(payload, expiry: DEFAULT_EXPIRY)
    payload[:exp] = expiry.from_now.to_i
    payload[:iat] = Time.current.to_i
    JWT.encode(payload, SECRET, ALGORITHM)
  end

  def self.decode(token)
    decoded = JWT.decode(token, SECRET, true, algorithm: ALGORITHM)
    HashWithIndifferentAccess.new(decoded.first)
  rescue JWT::ExpiredSignature
    raise JWT::DecodeError, "Token has expired"
  rescue JWT::DecodeError => e
    raise JWT::DecodeError, e.message
  end
end
```

## Auth Controller

```ruby
# app/controllers/api/v1/auth_controller.rb
class Api::V1::AuthController < Api::BaseController
  skip_before_action :authenticate, only: [:create, :refresh]

  # POST /api/v1/auth
  def create
    user = User.find_by(email: params[:email])

    if user&.authenticate(params[:password])
      tokens = generate_tokens(user)
      render json: {
        data: { user_id: user.id, email: user.email },
        meta: tokens
      }
    else
      render json: { error: "Invalid email or password" }, status: :unauthorized
    end
  end

  # POST /api/v1/auth/refresh
  def refresh
    payload = JwtService.decode(params[:refresh_token])
    user = User.find(payload[:user_id])
    tokens = generate_tokens(user)
    render json: { meta: tokens }
  end

  private

  def generate_tokens(user)
    {
      access_token: JwtService.encode(user_id: user.id),
      refresh_token: JwtService.encode({ user_id: user.id }, expiry: 30.days),
      expires_in: 24.hours.to_i
    }
  end
end
```

## Serializer Patterns

### ActiveModel::Serializer

```ruby
# app/serializers/product_serializer.rb
class ProductSerializer < ActiveModel::Serializer
  attributes :id, :name, :price, :description, :created_at, :updated_at

  has_many :reviews
  belongs_to :category

  def price
    "$#{'%.2f' % object.price}"
  end

  # Conditional inclusion
  attribute :admin_notes, if: :admin?

  def admin?
    scope&.admin?  # scope is current_user
  end
end
```

### Blueprinter

```ruby
# app/blueprints/product_blueprint.rb
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

# Usage in controller:
render json: ProductBlueprint.render(@product, view: :extended)
```

### Manual JBUILDER

```ruby
# app/views/api/v1/products/index.json.jbuilder
json.data @products do |product|
  json.id product.id
  json.type "products"
  json.attributes do
    json.name product.name
    json.price number_to_currency(product.price)
    json.description product.description
  end
  json.relationships do
    json.category do
      json.data { json.id product.category_id; json.type "categories" }
    end
  end
end

json.meta do
  json.total @products.total_count
  json.page @products.current_page
  json.per_page @products.limit_value
end
```

## JSON:API Compliant Response Structure

```json
{
  "data": [
    {
      "id": "1",
      "type": "products",
      "attributes": {
        "name": "Widget",
        "price": "$9.99",
        "description": "A fine widget"
      },
      "relationships": {
        "category": {
          "data": { "id": "5", "type": "categories" }
        },
        "reviews": {
          "data": [
            { "id": "10", "type": "reviews" },
            { "id": "11", "type": "reviews" }
          ]
        }
      }
    }
  ],
  "included": [
    {
      "id": "5",
      "type": "categories",
      "attributes": { "name": "Electronics" }
    }
  ],
  "meta": {
    "total": 42,
    "page": 1,
    "per_page": 20
  },
  "links": {
    "self": "/api/v1/products?page=1",
    "next": "/api/v1/products?page=2",
    "last": "/api/v1/products?page=3"
  }
}
```

## API Versioning

### URL Namespace (Recommended)

```ruby
# config/routes.rb
namespace :api do
  namespace :v1 do
    resources :products
    resources :users, only: [:index, :show]
    resource :auth, only: [:create], controller: "auth"
  end

  namespace :v2 do
    resources :products  # New serializer, same model
  end
end
```

### Header-Based Versioning (Alternative)

```ruby
# app/controllers/api/base_controller.rb
class Api::BaseController < ActionController::API
  before_action :set_api_version

  private

  def set_api_version
    @api_version = request.headers["API-Version"] || "v1"
  end

  def serializer_for(resource)
    version_module = @api_version.upcase.to_sym  # :V1, :V2
    "Api::#{version_module}::#{resource.class.name}Serializer".constantize
  end
end
```

## Rate Limiting

```ruby
# config/initializers/rack_attack.rb
class Rack::Attack
  # Per-token API rate limit
  throttle("api/token", limit: 100, period: 1.minute) do |req|
    if req.path.start_with?("/api/")
      req.env["HTTP_AUTHORIZATION"]&.split("Bearer ")&.last
    end
  end

  # Per-IP API rate limit (for unauthenticated endpoints)
  throttle("api/ip", limit: 20, period: 1.minute) do |req|
    req.ip if req.path.start_with?("/api/") && req.env["HTTP_AUTHORIZATION"].blank?
  end

  self.throttled_responder = lambda do |matched, env|
    now = Time.current
    headers = {
      "Content-Type" => "application/json",
      "Retry-After" => (now + matched[:period]).httpdate,
      "X-RateLimit-Limit" => matched[:limit].to_s,
      "X-RateLimit-Remaining" => "0"
    }
    body = { error: "Rate limit exceeded. Retry after #{matched[:period]} seconds." }.to_json
    [429, headers, [body]]
  end
end
```

## Pagination Helper

```ruby
# app/controllers/concerns/paginatable.rb
module Paginatable
  extend ActiveSupport::Concern

  DEFAULTS = { page: 1, per_page: 20, max_per_page: 100 }.freeze

  private

  def page_params
    page     = (params[:page] || DEFAULTS[:page]).to_i
    per_page = [(params[:per_page] || DEFAULTS[:per_page]).to_i, DEFAULTS[:max_per_page]].min
    { page: page, per_page: per_page }
  end

  def paginate(scope)
    pp = page_params
    scope.page(pp[:page]).per(pp[:per_page])
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

## CORS Configuration

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

## API Error Codes Convention

| HTTP Status | Meaning | When |
|-------------|---------|------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Missing/malformed params |
| 401 | Unauthorized | Missing/invalid auth token |
| 403 | Forbidden | Valid auth but insufficient permissions |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Stale update (optimistic locking) |
| 422 | Unprocessable Entity | Validation errors |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Unexpected server failure |
