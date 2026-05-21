# Axis A — Logic Placement

Where non-trivial business logic lives. Two values: `native` and `extracted`. Each section below is self-contained — read only the one that matches the resolved axis value.

---

## native

Logic lives in models, concerns, and plain Ruby objects. The "Rails Way" as advocated by DHH: trust Rails defaults, avoid unnecessary architectural layers, keep logic close to the data.

### Beliefs

- **Models are smart.** Business logic belongs in models — that is what object-orientation means.
- **Concerns decompose models** without introducing a new layer. Slice by trait or role: `Triageable`, `Postponable`, `Closeable`.
- **Callbacks are fine** for simple, predictable side effects ("whenever X happens, do Y"). Not everything needs a service object.
- **POROs for operations** that do not fit a model — plain objects, often in `app/models/`, not a `Service` suffix.

### Characteristic patterns

Business logic in models with concerns:

```ruby
# app/models/order.rb
class Order < ApplicationRecord
  include Purchasable
  include Trackable

  belongs_to :user
  has_many :line_items, dependent: :destroy

  validates :status, inclusion: { in: %w[pending confirmed shipped delivered] }
  scope :recent, -> { where("created_at > ?", 30.days.ago) }

  def total
    line_items.sum(&:subtotal) + tax + shipping
  end

  def confirm!
    update!(status: "confirmed", confirmed_at: Time.current)
    OrderMailer.confirmation(self).deliver_later
  end
end
```

```ruby
# app/models/concerns/purchasable.rb
module Purchasable
  extend ActiveSupport::Concern

  included do
    has_many :payments, as: :purchasable
    scope :paid, -> { joins(:payments).where(payments: { status: "completed" }) }
  end

  def paid?
    payments.completed.exists?
  end

  def outstanding_balance
    total - payments.completed.sum(:amount)
  end
end
```

### Extraction rule

Default to model methods and concerns. Extract a plain Ruby object only when an operation genuinely spans multiple unrelated models or an external system, and even then prefer a PORO over a `Service`-suffixed class.

### Detection signals

`app/services/` absent or near-empty; rich `app/models/concerns/`; callbacks orchestrating workflows; `gem "rails"` with few architectural gems.

---

## extracted

Service and command objects are the default home for non-trivial business logic. Favors explicit layers — common in larger teams, consultancy-built apps, and complex domains.

### Beliefs

- **Thin everything.** Controllers route, models validate, services orchestrate.
- **Service objects for workflows.** Any multi-step process gets its own class.
- **Result objects, not exceptions,** for expected flow control.
- **Composition over callbacks.** Side effects are explicit steps in a service, not hidden in model lifecycle hooks.

### Characteristic patterns

Service object with the Result pattern:

```ruby
# app/services/orders/create.rb
module Orders
  class Create
    def initialize(user:, params:)
      @user = user
      @params = params
    end

    def call
      order = @user.orders.build(@params)

      ActiveRecord::Base.transaction do
        order.save!
        Payments::Charge.new(order: order).call
        OrderMailer.confirmation(order).deliver_later
      end

      Result.new(success: true, order: order)
    rescue ActiveRecord::RecordInvalid => e
      Result.new(success: false, error: e.message, order: e.record)
    rescue Payments::ChargeError => e
      Result.new(success: false, error: e.message)
    end
  end
end
```

Thin controller delegating to the service:

```ruby
class OrdersController < ApplicationController
  def create
    result = Orders::Create.new(user: current_user, params: order_params).call

    if result.success?
      redirect_to result.order, notice: "Order placed!"
    else
      @order = result.order || Order.new(order_params)
      flash.now[:alert] = result.error
      render :new, status: :unprocessable_entity
    end
  end
end
```

### Extraction rule

Service objects are the default extraction target for any non-trivial business logic — domain logic for a single model, multi-model workflows, and external API calls alike.

### Detection signals

`app/services/` with several files; custom `ApplicationService` / `BaseService` base class; `gem "dry-monads"`, `gem "interactor"`, or `gem "trailblazer"`.
