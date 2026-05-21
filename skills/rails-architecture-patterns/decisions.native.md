# Architecture Decisions — `native` axis

Where logic lives, and how to fix structural smells, when the project keeps business logic in models and concerns.

## Where does logic go?

| Scenario | Home |
|----------|------|
| Calculates from own attributes | Model method |
| Shared behavior across 2+ models | Concern |
| Domain logic for one model | Concern or model method |
| Coordinates multiple models | Concern with callbacks, or a model method |
| Calls external API | Model method wrapping a client |
| Multi-step with rollback | Model method + transaction |
| Simple side effect (email, log) | Callback (`after_commit`) |

A concern can carry domain logic, not just behavior shared across models:

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

## God model fix

Symptom: model > 500 lines. Extract to concerns sliced by responsibility:

```ruby
class User < ApplicationRecord
  include Authenticatable    # Auth logic
  include Billable           # Subscription/payment
  include Notifiable         # Notification preferences
end
```

## Fat controller fix

Symptom: controller action > 15 lines. Enrich the model; the controller calls one model method:

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

## Callback hell fix

Symptom: long chains of `after_create` callbacks. Keep callbacks for data integrity and simple expected effects; put multi-step workflows in explicit methods:

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
