---
name: rails-mailer-patterns
description: Action Mailer patterns for Rails applications. Automatically invoked when working with email delivery, mailer classes, email templates, mailer previews, interceptors, or delivery configuration. Triggers on "mailer", "email", "ActionMailer", "deliver_later", "deliver_now", "mail template", "email preview", "SMTP", "SendGrid", "Postmark", "notification email". NOT for push notifications, SMS, or in-app messaging.
allowed-tools: Read, Grep, Glob
---

# Rails Mailer Patterns

Analyze and recommend Action Mailer patterns for reliable, well-structured email delivery in Rails applications.

See [patterns.md](patterns.md) for detailed code examples.

## Quick Reference

| Pattern | Use When | Example |
|---------|----------|---------|
| `deliver_later` | Default for all emails | `UserMailer.welcome(user).deliver_later` |
| `deliver_now` | Must send immediately (e.g., password reset in test) | `UserMailer.reset(user).deliver_now` |
| Parameterized mailer | Shared context across methods | `UserMailer.with(user: user).welcome` |
| Mailer preview | Visual email testing | `test/mailers/previews/` or `spec/mailers/previews/` |
| Interceptor | Modify all outgoing email (staging redirect) | `register_interceptor(StagingInterceptor)` |

## Core Principles

1. **Always `deliver_later`**: Email delivery is I/O — send via background job by default
2. **Mailers are views, not logic**: Mailer methods assemble data and call `mail()`. No business logic
3. **One email per method**: Each mailer method sends one email. No conditionals choosing between templates
4. **Preview every email**: Use mailer previews for visual verification — they catch layout issues CI can't
5. **Test content, not delivery**: Assert email content and recipients; trust Rails for delivery mechanics

## Mailer Structure Template

```ruby
class OrderMailer < ApplicationMailer
  def confirmation(order)
    @order = order
    @user = order.user

    mail(
      to: @user.email,
      subject: "Order ##{@order.number} confirmed"
    )
  end

  def shipped(order)
    @order = order
    @tracking_url = order.tracking_url

    mail(to: order.user.email)
    # Subject from: app/views/order_mailer/shipped.en.yml
  end
end
```

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

## Application Mailer Base

```ruby
class ApplicationMailer < ActionMailer::Base
  default from: -> { "#{Current.account&.name || 'App'} <noreply@example.com>" }
  layout "mailer"

  # Omakase: rescue delivery errors in mailer
  rescue_from Net::SMTPSyntaxError, with: :log_delivery_error

  private

  def log_delivery_error(exception)
    Rails.logger.error("[Mailer] Delivery failed: #{exception.message}")
  end
end
```

## Previews

```ruby
# test/mailers/previews/order_mailer_preview.rb (Omakase)
# spec/mailers/previews/order_mailer_preview.rb (Service-oriented)
class OrderMailerPreview < ActionMailer::Preview
  def confirmation
    order = Order.first || FactoryBot.build(:order) # Adapt to profile
    OrderMailer.confirmation(order)
  end

  def shipped
    order = Order.where.not(shipped_at: nil).first
    OrderMailer.shipped(order)
  end
end
# Visit: http://localhost:3000/rails/mailers/order_mailer/confirmation
```

## Testing

**Omakase — Minitest:**

```ruby
class OrderMailerTest < ActionMailer::TestCase
  test "confirmation email" do
    order = orders(:confirmed)
    email = OrderMailer.confirmation(order)

    assert_emails 1 do
      email.deliver_now
    end
    assert_equal ["noreply@example.com"], email.from
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

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| `deliver_now` in controllers | Blocks request, slow response | Use `deliver_later` |
| Business logic in mailers | Wrong layer, hard to test | Move to model/service, pass data to mailer |
| Conditional templates in one method | Hard to test and preview | One method per email |
| No previews | Layout bugs found in production | Add preview for every mailer method |
| Hardcoded from address | Can't change per environment | Use `default from:` in ApplicationMailer |
| HTML-only emails | Spam filters, accessibility | Always include text part |

## Output Format

When creating mailers, provide:
1. **Mailer class** with proper structure
2. **View templates** (HTML + text parts)
3. **Preview class** for visual testing
4. **Test file** matching project profile (Minitest or RSpec)
5. **Delivery configuration** notes if relevant
