# Action Mailer Patterns — Detailed Reference

Follow standard Rails conventions for basic mailer structure, SMTP configuration, multipart templates, attachments, I18n subjects, and `letter_opener` in development. This file covers opinionated and non-obvious patterns.

## Parameterized Mailers with `before_action`

The key pattern: use `params` and `before_action` to share context across methods without repeating arguments.

```ruby
class UserMailer < ApplicationMailer
  before_action { @user = params[:user] }
  before_action { @account = params[:user].account }

  default to: -> { @user.email }

  def welcome
    mail(subject: "Welcome to #{@account.name}")
  end

  def weekly_digest
    @events = @user.events.from_last_week
    mail(subject: "Your weekly digest")
  end
end

# Usage — always with(params).method
UserMailer.with(user: user).welcome.deliver_later
```

## ApplicationMailer with Dynamic From and Error Handling

```ruby
class ApplicationMailer < ActionMailer::Base
  default from: -> { "#{Current.account&.name || 'App'} <noreply@example.com>" }
  layout "mailer"

  rescue_from Net::SMTPSyntaxError, with: :log_delivery_error

  before_action :set_default_headers

  private

  def set_default_headers
    headers["X-App-Version"] = Rails.application.config.version
    headers["List-Unsubscribe"] = "<mailto:unsubscribe@example.com>"
  end

  def log_delivery_error(exception)
    Rails.logger.error("[Mailer] Delivery failed: #{exception.message}")
  end
end
```

## Staging Email Interceptor

Prevents real emails in staging by redirecting all mail:

```ruby
# app/mailers/interceptors/staging_interceptor.rb
class StagingInterceptor
  def self.delivering_email(message)
    message.to = ["staging-inbox@example.com"]
    message.cc = nil
    message.bcc = nil
    message.subject = "[STAGING] #{message.subject} (was: #{message.to})"
  end
end

# config/initializers/mail_interceptors.rb
if Rails.env.staging?
  ActionMailer::Base.register_interceptor(StagingInterceptor)
end
```

## Delivery Observer (Logging/Tracking)

```ruby
class DeliveryObserver
  def self.delivered_email(message)
    EmailLog.create!(
      to: message.to.join(", "),
      subject: message.subject,
      delivered_at: Time.current
    )
  end
end

ActionMailer::Base.register_observer(DeliveryObserver)
```

## Previews with Multiple Scenarios

```ruby
class OrderMailerPreview < ActionMailer::Preview
  def confirmation
    OrderMailer.confirmation(Order.first)
  end

  def confirmation_with_discount
    order = Order.joins(:discount).first
    OrderMailer.confirmation(order)
  end

  def confirmation_international
    order = Order.joins(:user).where(users: { locale: "ja" }).first
    OrderMailer.confirmation(order)
  end
end
```

## Conditional Delivery (Skip Empty)

```ruby
class NotificationMailer < ApplicationMailer
  def activity_digest(user)
    @user = user
    @activities = user.activities.from_last_day
    return if @activities.none?  # Returning nil skips delivery entirely

    mail(to: @user.email, subject: "Your daily activity")
  end
end
```

## Profile-Aware Delivery Triggers

**Omakase — model callback triggers delivery (uses `_later`/`_now` convention):**

```ruby
class Order < ApplicationRecord
  after_create_commit :send_confirmation

  private

  def send_confirmation
    OrderMailer.confirmation(self).deliver_later
  end
end
```

**Service-oriented — service triggers delivery:**

```ruby
class Orders::Create
  def call
    order = Order.create!(params)
    OrderMailer.confirmation(order).deliver_later
    Result.new(success: true, order: order)
  end
end
```

## Queue Configuration

```ruby
class ApplicationMailer < ActionMailer::Base
  self.deliver_later_queue_name = :mailers
end

class UrgentMailer < ApplicationMailer
  self.deliver_later_queue_name = :critical
end
```

## Profile-Aware Testing

**Omakase — Minitest:**

```ruby
class OrderMailerTest < ActionMailer::TestCase
  test "confirmation email" do
    order = orders(:confirmed)
    email = OrderMailer.confirmation(order)

    assert_emails 1 do
      email.deliver_now
    end
    assert_equal [order.user.email], email.to
    assert_match "Order ##{order.number}", email.subject
  end
end
```

**Service-oriented — RSpec with `have_enqueued_mail`:**

```ruby
RSpec.describe "Order creation", type: :request do
  it "sends confirmation email" do
    expect {
      post orders_path, params: { order: attributes_for(:order) }
    }.to have_enqueued_mail(OrderMailer, :confirmation)
  end
end
```
