# Architecture Decisions — `extracted` axis

Where logic lives, and how to fix structural smells, when service objects are the default home for business logic.

## Where does logic go?

| Scenario | Home |
|----------|------|
| Calculates from own attributes | Model method |
| Shared behavior across 2+ models | Concern |
| Domain logic for one model | Service object |
| Coordinates multiple models | Service object |
| Calls external API | Service object |
| Multi-step with rollback | Service object + transaction |
| Simple side effect (email, log) | Service object |

## God model fix

Symptom: model > 500 lines. Extract to service objects:

```
app/services/
  users/
    registration_service.rb
    billing_service.rb
    notification_service.rb
```

## Fat controller fix

Symptom: controller action > 15 lines. Extract to a service; the controller calls it and handles the Result:

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

## Callback hell fix

Symptom: long chains of `after_create` callbacks. Callbacks guard data integrity only; the workflow lives in a service:

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
