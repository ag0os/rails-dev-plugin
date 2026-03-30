# Architecture Patterns — Decision Frameworks

## STI vs Polymorphic Associations

**Decision checklist with specific thresholds:**

| Factor | STI | Polymorphic |
|--------|-----|-------------|
| Shared columns > 70% | Yes | -- |
| Shared columns < 30% | -- | Yes |
| Need to query all types together | Yes | Possible but slower |
| Types have different associations | -- | Yes |
| Type-specific columns | Few (< 5 nullable) | Many |

Follow standard Rails conventions for implementing whichever you choose.

## Service Object vs Concern vs Model Method (Profile-Dependent)

**The right answer depends on the project's stack profile.**

**Decision matrix:**

| Scenario | Omakase | Service-Oriented |
|----------|---------|-----------------|
| Calculates from own attributes | Model method | Model method |
| Shared behavior across 2+ models | Concern | Concern |
| Domain logic for one model | Concern or model method | Service object |
| Coordinates multiple models | Concern with callbacks, or model method | Service object |
| Calls external API | Model method wrapping client | Service object |
| Multi-step with rollback | Model method + transaction | Service object + transaction |
| Simple side effect (email, log) | Callback (`after_commit`) | Service object |

**Omakase pattern -- concern for domain logic (not just shared behavior):**

```ruby
# app/models/concerns/purchasable.rb
module Purchasable
  extend ActiveSupport::Concern

  included do
    has_many :payments, as: :purchasable
    after_create_commit :send_confirmation
  end

  def charge!
    PaymentGateway.charge(total, user.payment_method)
    update!(paid_at: Time.current)
  end
end
```

## God Model Fix (Profile-Dependent)

**Symptom:** Model > 500 lines, too many responsibilities.

**Omakase fix -- extract to concerns:**

```ruby
class User < ApplicationRecord
  include Authenticatable    # Auth logic
  include Billable           # Subscription/payment
  include Notifiable         # Notification preferences
end
```

**Service-oriented fix -- extract to service objects:**

```
app/services/
  users/
    registration_service.rb
    billing_service.rb
    notification_service.rb
```

## Fat Controller Fix (Profile-Dependent)

**Symptom:** Controller actions > 15 lines, business logic in controller.

**Omakase fix -- enrich the model:**

```ruby
def create
  @order = current_user.orders.build(order_params)
  if @order.place!  # Model method handles tax, discount, payment, email
    redirect_to @order
  else
    render :new, status: :unprocessable_entity
  end
end
```

**Service-oriented fix -- extract to service:**

```ruby
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

## Callback Hell Fix (Profile-Dependent)

**Symptom:** Long chains of `after_create` callbacks triggering side effects.

**Omakase fix -- callbacks for data integrity, explicit methods for workflows:**

```ruby
class Order < ApplicationRecord
  before_validation :normalize_status
  after_create_commit :send_confirmation_email  # Simple, expected

  def place!
    transaction do
      save!
      charge_payment
      reserve_inventory
    end
  end
end
```

**Service-oriented fix -- callbacks only for data integrity:**

```ruby
class Order < ApplicationRecord
  before_validation :normalize_status
  after_create :set_order_number  # Pure data concern
end

# Workflow in service
class PlaceOrderService
  def call
    order = Order.create!(params)
    PaymentService.charge(order)
    OrderMailer.confirmation(order).deliver_later
    InventoryService.reserve(order)
  end
end
```

## Monolith vs Engine vs Microservice

| Factor | Monolith | Engine | Microservice |
|--------|----------|--------|-------------|
| Team size | 1-10 | 5-20 | 10+ |
| Deploy independently | No | No | Yes |
| Code isolation | Directories | Gem boundary | Network boundary |
| Complexity | Low | Medium | High |
| When to choose | Default (always start here) | Large monolith needs internal boundaries | Proven need for independent scaling |

**Rule:** Start with a monolith. Only extract an engine when you have clear domain boundaries and shared code between apps. Only extract a microservice when you have a proven scaling bottleneck that can't be solved within the monolith.

## Premature Optimization Rule

```
1. First make it work (plain Rails)
2. Then make it right (refactor with data)
3. Then make it fast (optimize with measurements)
```

Don't add caching, service objects, engines, or microservices before measuring. YAGNI applies to architecture too.
