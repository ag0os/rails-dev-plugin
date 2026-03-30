---
name: rails-mailer-patterns
description: Action Mailer patterns for Rails applications. Automatically invoked when working with email delivery, mailer classes, email templates, mailer previews, interceptors, or delivery configuration. Triggers on "mailer", "email", "ActionMailer", "deliver_later", "deliver_now", "mail template", "email preview", "SMTP", "SendGrid", "Postmark", "notification email". NOT for push notifications, SMS, or in-app messaging.
allowed-tools: Read, Grep, Glob
---

# Rails Mailer Patterns

Analyze and recommend Action Mailer patterns for reliable, well-structured email delivery in Rails applications.

Follow standard Rails conventions for basic mailer structure, `mail()` calls, delivery configuration, multipart templates, attachments, and I18n subjects. Focus on the opinionated patterns below.

See [patterns.md](patterns.md) for detailed code examples.

## Quick Reference

| Pattern | Use When |
|---------|----------|
| Parameterized mailer with `before_action` | Shared context across multiple mailer methods |
| Mailer preview per method | Every mailer method needs a preview — catches layout issues CI cannot |
| Staging interceptor | Prevent real emails in non-production environments |
| `_later`/`_now` convention | Model triggers delivery; mailer is just the template layer |
| Profile-aware testing | Minitest (omakase) vs RSpec (service-oriented) |

## Core Principles

1. **Always `deliver_later`**: Email delivery is I/O — background job by default
2. **Mailers are views, not logic**: Assemble data and call `mail()`. No business logic
3. **One email per method**: No conditionals choosing between templates
4. **Preview every email**: Mailer previews catch layout issues CI cannot
5. **`_later`/`_now` on the model**: The model defines when to send; the mailer defines what to send

## Parameterized Mailers

Share context across methods without repeating arguments:

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

# Usage
UserMailer.with(user: user).welcome.deliver_later
```

## ApplicationMailer with Dynamic From

```ruby
class ApplicationMailer < ActionMailer::Base
  default from: -> { "#{Current.account&.name || 'App'} <noreply@example.com>" }
  layout "mailer"

  rescue_from Net::SMTPSyntaxError, with: :log_delivery_error

  private

  def log_delivery_error(exception)
    Rails.logger.error("[Mailer] Delivery failed: #{exception.message}")
  end
end
```

## Staging Email Interceptor

Prevent sending real emails in staging — redirects all mail to a safe inbox:

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

## Previews

```ruby
# test/mailers/previews/order_mailer_preview.rb  (Omakase)
# spec/mailers/previews/order_mailer_preview.rb   (Service-oriented)
class OrderMailerPreview < ActionMailer::Preview
  def confirmation
    order = Order.first || FactoryBot.build(:order)
    OrderMailer.confirmation(order)
  end

  def confirmation_with_discount
    order = Order.joins(:discount).first
    OrderMailer.confirmation(order)
  end
end
# Visit: http://localhost:3000/rails/mailers/order_mailer/confirmation
```

## Profile-Aware Delivery Triggers

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

**Service-oriented — RSpec:**

```ruby
RSpec.describe OrderMailer, type: :mailer do
  describe "#confirmation" do
    let(:order) { create(:order) }
    let(:mail) { described_class.confirmation(order) }

    it "renders the headers" do
      expect(mail.to).to eq([order.user.email])
      expect(mail.subject).to match(/Order ##{order.number}/)
    end

    it "renders the body" do
      expect(mail.body.encoded).to include(order.number)
    end
  end
end
```

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| `deliver_now` in controllers | Use `deliver_later` — email is I/O |
| Business logic in mailers | Move to model/service, pass data to mailer |
| Conditional templates in one method | One method per email |
| No previews | Add preview for every mailer method |
| HTML-only emails | Always include text part |

## Output Format

When creating mailers, provide:
1. **Mailer class** with proper structure
2. **View templates** (HTML + text parts)
3. **Preview class** for visual testing
4. **Test file** matching project profile (Minitest or RSpec)
5. **Delivery configuration** notes if relevant
