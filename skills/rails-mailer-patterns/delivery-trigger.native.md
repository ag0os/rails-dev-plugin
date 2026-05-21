# Mailer Delivery Triggers — `native` axis

A model callback triggers delivery, following the `_later`/`_now` convention. The model decides *when* to send; the mailer only defines *what* to send.

```ruby
class Order < ApplicationRecord
  after_create_commit :send_confirmation

  private

  def send_confirmation
    OrderMailer.confirmation(self).deliver_later
  end
end
```

Keep the trigger callback simple — one `deliver_later` call. If choosing whether to send involves real branching logic, move that decision into an explicit model method rather than growing the callback.
