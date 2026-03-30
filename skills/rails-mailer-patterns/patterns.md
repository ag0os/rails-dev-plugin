# Action Mailer Patterns — Detailed Reference

## Delivery Configuration

### Production (common providers)

```ruby
# config/environments/production.rb

# SMTP (SendGrid, Postmark, Mailgun)
config.action_mailer.delivery_method = :smtp
config.action_mailer.smtp_settings = {
  address: "smtp.sendgrid.net",
  port: 587,
  user_name: "apikey",
  password: Rails.application.credentials.dig(:sendgrid, :api_key),
  authentication: :plain,
  enable_starttls_auto: true
}

# Default URL for links in emails
config.action_mailer.default_url_options = { host: "example.com", protocol: "https" }
config.action_mailer.asset_host = "https://example.com"
```

### Development

```ruby
# config/environments/development.rb
config.action_mailer.delivery_method = :letter_opener # gem "letter_opener"
config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
config.action_mailer.raise_delivery_errors = true
config.action_mailer.perform_deliveries = true
```

### Test

```ruby
# config/environments/test.rb
config.action_mailer.delivery_method = :test
config.action_mailer.default_url_options = { host: "localhost", port: 3000 }
```

## Multipart Emails (HTML + Text)

Always provide both HTML and text versions:

```erb
<%# app/views/order_mailer/confirmation.html.erb %>
<h1>Order Confirmed</h1>
<p>Hi <%= @user.name %>,</p>
<p>Your order <strong>#<%= @order.number %></strong> has been confirmed.</p>
<%= link_to "View Order", order_url(@order) %>
```

```erb
<%# app/views/order_mailer/confirmation.text.erb %>
Order Confirmed

Hi <%= @user.name %>,

Your order #<%= @order.number %> has been confirmed.

View Order: <%= order_url(@order) %>
```

## Mailer Layout

```erb
<%# app/views/layouts/mailer.html.erb %>
<!DOCTYPE html>
<html>
<head>
  <meta content="text/html; charset=UTF-8" http-equiv="Content-Type" />
  <style>
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; }
    .container { max-width: 600px; margin: 0 auto; padding: 20px; }
    .footer { margin-top: 30px; font-size: 12px; color: #666; }
  </style>
</head>
<body>
  <div class="container">
    <%= yield %>
    <div class="footer">
      <p>&copy; <%= Date.current.year %> <%= @account&.name || "App" %></p>
      <%= link_to "Unsubscribe", unsubscribe_url(token: @user&.unsubscribe_token) %>
    </div>
  </div>
</body>
</html>
```

## Attachments

```ruby
class InvoiceMailer < ApplicationMailer
  def send_invoice(invoice)
    @invoice = invoice

    # File attachment
    attachments["invoice_#{invoice.number}.pdf"] = invoice.generate_pdf

    # Inline attachment (for embedded images)
    attachments.inline["logo.png"] = File.read(Rails.root.join("app/assets/images/logo.png"))

    mail(to: invoice.user.email, subject: "Invoice ##{invoice.number}")
  end
end
```

```erb
<%# Reference inline attachment in template %>
<%= image_tag attachments["logo.png"].url %>
```

## Interceptors and Observers

### Staging Email Redirect

Prevent sending real emails in staging:

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

### Delivery Observer (logging, tracking)

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

## Conditional Delivery

```ruby
class NotificationMailer < ApplicationMailer
  def activity_digest(user)
    @user = user
    @activities = user.activities.from_last_day

    # Don't send empty digests
    return if @activities.none?

    mail(to: @user.email, subject: "Your daily activity")
  end
end
```

**Note:** Returning `nil` from a mailer method skips delivery entirely.

## Callbacks

```ruby
class ApplicationMailer < ActionMailer::Base
  before_action :set_default_headers
  after_action :log_delivery

  private

  def set_default_headers
    headers["X-App-Version"] = Rails.application.config.version
    headers["List-Unsubscribe"] = "<mailto:unsubscribe@example.com>"
  end

  def log_delivery
    Rails.logger.info("[Mailer] Preparing: #{self.class}##{action_name} to #{message.to}")
  end
end
```

## Mailer with Background Job Integration

**Omakase — model triggers delivery:**

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

# Or per mailer
class UrgentMailer < ApplicationMailer
  self.deliver_later_queue_name = :critical
end
```

## I18n in Emails

```yaml
# config/locales/en.yml
en:
  order_mailer:
    confirmation:
      subject: "Order #%{number} confirmed"
    shipped:
      subject: "Your order has shipped!"
```

```ruby
class OrderMailer < ApplicationMailer
  def confirmation(order)
    @order = order
    mail(
      to: order.user.email,
      subject: default_i18n_subject(number: order.number)
    )
  end
end
```

## Previews with Different Scenarios

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

## Testing Delivery in Integration Tests

**Omakase:**

```ruby
class OrdersControllerTest < ActionDispatch::IntegrationTest
  test "creating order sends confirmation email" do
    assert_emails 1 do
      post orders_url, params: { order: { product_id: products(:widget).id } }
    end
  end

  test "confirmation email has correct content" do
    post orders_url, params: { order: { product_id: products(:widget).id } }

    email = ActionMailer::Base.deliveries.last
    assert_equal [users(:jane).email], email.to
    assert_match /confirmed/i, email.subject
  end
end
```

**Service-oriented:**

```ruby
RSpec.describe "Order creation", type: :request do
  it "sends confirmation email" do
    expect {
      post orders_path, params: { order: attributes_for(:order) }
    }.to have_enqueued_mail(OrderMailer, :confirmation)
  end
end
```

## Directory Organization

```
app/
  mailers/
    application_mailer.rb
    order_mailer.rb
    user_mailer.rb
    notification_mailer.rb
    interceptors/              # Custom interceptors
      staging_interceptor.rb
  views/
    layouts/
      mailer.html.erb          # Shared HTML layout
      mailer.text.erb          # Shared text layout
    order_mailer/
      confirmation.html.erb
      confirmation.text.erb
      shipped.html.erb
      shipped.text.erb
    user_mailer/
      welcome.html.erb
      welcome.text.erb
test/mailers/                  # Omakase
  order_mailer_test.rb
  previews/
    order_mailer_preview.rb
spec/mailers/                  # Service-oriented
  order_mailer_spec.rb
  previews/
    order_mailer_preview.rb
```
