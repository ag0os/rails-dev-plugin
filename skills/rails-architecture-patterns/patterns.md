# Rails Architecture Patterns -- Detailed Reference

## Decision Frameworks

### STI vs Polymorphic Associations

**Single Table Inheritance (STI)**

Use when types share most columns and behavior:

```ruby
# One table: vehicles (type, make, model, year, doors, payload_capacity)
class Vehicle < ApplicationRecord
  validates :make, :model, :year, presence: true
end

class Car < Vehicle
  validates :doors, presence: true
end

class Truck < Vehicle
  validates :payload_capacity, presence: true
end
```

**Polymorphic Associations**

Use when types have different attributes and belong to unrelated models:

```ruby
# Separate tables, shared interface
class Comment < ApplicationRecord
  belongs_to :commentable, polymorphic: true
end

class Article < ApplicationRecord
  has_many :comments, as: :commentable
end

class Photo < ApplicationRecord
  has_many :comments, as: :commentable
end
```

**Decision checklist:**

| Factor | STI | Polymorphic |
|--------|-----|-------------|
| Shared columns > 70% | Yes | -- |
| Shared columns < 30% | -- | Yes |
| Need to query all types together | Yes | Possible but slower |
| Types have different associations | -- | Yes |
| Number of type-specific columns | Few (< 5) | Many |
| Null columns acceptable | Yes | Avoid |

### Service Object vs Concern vs Plain Model Method

**The right answer depends on the project's stack profile.** See `rails-stack-profiles` for detection.

**Plain Model Method**

Use for logic that operates on a single record's own data (all profiles):

```ruby
class Order < ApplicationRecord
  def total
    line_items.sum(&:subtotal)
  end
end
```

**Concern**

Use to share capabilities across multiple models. **Omakase profile:** concerns are also the primary way to decompose large models and encapsulate domain logic.

```ruby
# app/models/concerns/searchable.rb
module Searchable
  extend ActiveSupport::Concern

  included do
    scope :search, ->(query) {
      where("name ILIKE :q OR description ILIKE :q", q: "%#{query}%")
    }
  end
end
```

**Omakase pattern — concern for domain logic (not just shared behavior):**

```ruby
# app/models/concerns/purchasable.rb
module Purchasable
  extend ActiveSupport::Concern

  included do
    has_many :payments, as: :purchasable
    after_create :send_confirmation
  end

  def charge!
    PaymentGateway.charge(total, user.payment_method)
    update!(paid_at: Time.current)
  end

  private

  def send_confirmation
    OrderMailer.confirmation(self).deliver_later
  end
end
```

**Service Object**

Use for multi-step workflows that coordinate multiple models/external services. **Service-oriented and api-first profiles:** this is the default extraction pattern. **Omakase profile:** use only when a workflow genuinely spans multiple unrelated models or external systems.

```ruby
# app/services/checkout_service.rb
class CheckoutService
  def initialize(cart:, payment_method:)
    @cart = cart
    @payment_method = payment_method
  end

  def call
    ActiveRecord::Base.transaction do
      order = create_order
      charge = process_payment(order)
      send_confirmation(order)
      Result.new(success: true, order: order)
    end
  rescue PaymentError => e
    Result.new(success: false, error: e.message)
  end
end
```

**Decision matrix:**

| Scenario | Omakase | Service-Oriented |
|----------|---------|-----------------|
| Calculates from own attributes | Model method | Model method |
| Shared behavior across 2+ models | Concern | Concern |
| Domain logic for one model | Concern or model method | Service object |
| Coordinates multiple models | Concern with callbacks, or model method | Service object |
| Calls external API | Model method wrapping client | Service object |
| Multi-step with rollback | Model method + transaction | Service object + transaction |
| Simple side effect (email, log) | Callback | Service object |

### Monolith vs Engine vs Microservice

| Factor | Monolith | Engine | Microservice |
|--------|----------|--------|-------------|
| Team size | 1-10 | 5-20 | 10+ |
| Deploy independently | No | No | Yes |
| Code isolation | Directories | Gem boundary | Network boundary |
| Complexity | Low | Medium | High |
| When to choose | Default | Large monolith needs boundaries | Proven need for independent scaling |

**Rails Engine example:**

```ruby
# engines/billing/lib/billing/engine.rb
module Billing
  class Engine < ::Rails::Engine
    isolate_namespace Billing
  end
end
```

### When to Use Background Jobs

| Use a Job When | Keep Inline When |
|----------------|-----------------|
| External API call | Database read/write < 100ms |
| Email/notification sending | Simple validation |
| File processing/upload | Redirect after save |
| Report generation | Flash message display |
| Bulk data operations | Single record CRUD |
| Anything > 200ms | Must see result immediately |

### Database Design Decisions

**When to add an index:**

```ruby
# Always index:
# - Foreign keys
# - Columns in WHERE clauses
# - Columns in ORDER BY
# - Columns used in UNIQUE constraints

class AddIndexesToOrders < ActiveRecord::Migration[7.1]
  def change
    add_index :orders, :user_id
    add_index :orders, :status
    add_index :orders, [:user_id, :created_at]  # Composite for user's recent orders
    add_index :orders, :email, unique: true
  end
end
```

**Denormalization decision:**

| Factor | Normalize | Denormalize |
|--------|-----------|-------------|
| Write frequency | High | Low |
| Read frequency | Low | High |
| Data consistency critical | Yes | Less critical |
| Query complexity | Acceptable | Too many joins |
| Counter/sum needed frequently | -- | Use counter_cache |

```ruby
# Counter cache example
class Review < ApplicationRecord
  belongs_to :product, counter_cache: true
end
# Requires: products.reviews_count column
```

## Architecture Anti-Patterns

### God Model

**Symptom:** Model > 500 lines, handles too many responsibilities.

**Fix:** Extract concerns, service objects, and value objects.

```ruby
# Before: User model does everything
class User < ApplicationRecord
  # 600 lines of auth, billing, notifications, preferences...
end

# After: Extract into focused modules
class User < ApplicationRecord
  include Authenticatable    # Auth logic
  include Billable           # Subscription/payment
  include Notifiable         # Notification preferences
end
```

### Fat Controller

**Symptom:** Controller actions > 15 lines, business logic in controller.

**Fix depends on profile:**

**Omakase fix — move logic to the model:**

```ruby
# Before
def create
  @order = Order.new(order_params)
  @order.calculate_tax
  @order.apply_discount(current_user.discount_tier)
  if @order.save
    PaymentGateway.charge(@order.total, current_user.payment_method)
    OrderMailer.confirmation(@order).deliver_later
    redirect_to @order
  else
    render :new
  end
end

# After (omakase) — model handles its own workflow
def create
  @order = current_user.orders.build(order_params)
  if @order.place!  # Model method handles tax, discount, payment, email
    redirect_to @order
  else
    render :new, status: :unprocessable_entity
  end
end
```

**Service-oriented fix — extract to service object:**

```ruby
# After (service-oriented) — service orchestrates the workflow
def create
  result = CreateOrderService.call(params: order_params, user: current_user)
  if result.success?
    redirect_to result.order
  else
    @order = result.order
    render :new, status: :unprocessable_entity
  end
end
```

### Callback Hell

**Symptom:** Long chains of `after_create`, `before_save` callbacks that trigger side effects.

**Fix depends on profile:**

**Omakase fix — reduce to essential callbacks, use explicit model methods for workflows:**

```ruby
# Bad: Too many implicit callbacks
class Order < ApplicationRecord
  after_create :charge_payment
  after_create :send_confirmation_email
  after_create :update_inventory
  after_create :notify_warehouse
end

# Good (omakase): Callbacks for data integrity; explicit method for workflow
class Order < ApplicationRecord
  before_validation :normalize_status
  after_create_commit :send_confirmation_email  # Simple, expected side effect

  def place!
    transaction do
      save!
      charge_payment
      reserve_inventory
    end
  end
end
```

**Service-oriented fix — extract to service object:**

```ruby
# Good (service-oriented): Callbacks only for data integrity
class Order < ApplicationRecord
  before_validation :normalize_status
  after_create :set_order_number  # Pure data concern
end

# Workflow in service object
class PlaceOrderService
  def call
    order = Order.create!(params)
    PaymentService.charge(order)
    OrderMailer.confirmation(order).deliver_later
    InventoryService.reserve(order)
  end
end
```

### Premature Optimization

**Symptom:** Adding caching, service objects, engines before measuring.

**Rule of thumb:**
- First make it work (plain Rails)
- Then make it right (refactor)
- Then make it fast (optimize with data)

### Missing Abstraction Layer

**Symptom:** Controllers orchestrate complex multi-model operations directly.

**Fix depends on profile:**

**Omakase fix — enrich models with domain methods and concerns:**

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  include Purchasable    # Payment logic
  include Fulfillable    # Inventory + shipping logic

  def place!
    transaction do
      save!
      charge!
      reserve_inventory!
      OrderMailer.confirmation(self).deliver_later
    end
  end

  def cancel!
    transaction do
      refund!
      release_inventory!
      update!(status: "cancelled")
    end
  end
end
```

**Service-oriented fix — introduce service objects at the right granularity:**

```
app/services/
  orders/
    create_service.rb       # Creates order + payment + notifications
    cancel_service.rb       # Cancels + refund + inventory release
  users/
    registration_service.rb # Signup + welcome email + default settings
```

## Layer Responsibilities

| Layer | Should Do | Should NOT Do |
|-------|-----------|---------------|
| **Model** | Validations, associations, scopes, data methods | HTTP concerns, rendering, external API calls |
| **Controller** | Params, auth, call service, render response | Business logic, complex queries, email sending |
| **View** | Display data, layout, presentation helpers | Query database, mutate state |
| **Service** | Orchestrate workflows, coordinate models | Know about HTTP, render views |
| **Job** | Async execution of services | Complex business logic (delegate to service) |
| **Mailer** | Email templates and delivery | Business logic, database queries beyond lookup |

## File Organization for Large Apps

```
app/
  controllers/
    api/
      v1/           # Versioned API controllers
    admin/           # Admin namespace
    concerns/        # Shared controller concerns
  models/
    concerns/        # Shared model concerns
  services/
    orders/          # Domain-grouped services
    users/
    payments/
  forms/             # Form objects (multi-model forms)
  queries/           # Complex query objects
  presenters/        # View-specific logic
  policies/          # Authorization (Pundit)
  validators/        # Custom validators
```
