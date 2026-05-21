# Mailer Delivery Triggers — `extracted` axis

The service object that performs the operation triggers delivery as an explicit step in the workflow. Delivery is a visible line in the service, not a hidden model callback.

```ruby
class Orders::Create
  def call
    order = Order.create!(params)
    OrderMailer.confirmation(order).deliver_later
    Result.new(success: true, order: order)
  end
end
```

Because delivery is explicit, it is easy to see in code review which workflows send mail and to test the enqueue at the service boundary.
