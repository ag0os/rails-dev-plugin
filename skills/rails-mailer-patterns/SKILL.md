---
name: rails-mailer-patterns
description: Action Mailer patterns for Rails applications. Automatically invoked when working with email delivery, mailer classes, email templates, mailer previews, interceptors, or delivery configuration. Triggers on "mailer", "email", "ActionMailer", "deliver_later", "deliver_now", "mail template", "email preview", "SMTP", "SendGrid", "Postmark", "notification email". NOT for push notifications, SMS, or in-app messaging.
allowed-tools: Read, Grep, Glob
---

# Rails Mailer Patterns

Analyze and recommend Action Mailer patterns for reliable, well-structured email delivery in Rails applications.

Follow standard Rails conventions for basic mailer structure, `mail()` calls, delivery configuration, multipart templates, attachments, and I18n subjects. Focus on the opinionated patterns below.

## Supporting Documentation

- [patterns.md](patterns.md) - detailed code examples
- [delivery-trigger.native.md](delivery-trigger.native.md) / [delivery-trigger.extracted.md](delivery-trigger.extracted.md) - what triggers delivery, per Axis A

## Resolve the axis first

What triggers email delivery forks on **Axis A — logic placement** (`rails-stack-profiles`). Resolve it (or reuse the session value), then read the matching delivery-trigger guide above. `native` if it cannot be resolved.

## Quick Reference

| Pattern | Use When |
|---------|----------|
| Parameterized mailer with `before_action` | Shared context across multiple mailer methods |
| Mailer preview per method | Every mailer method needs a preview — catches layout issues CI cannot |
| Staging interceptor | Prevent real emails in non-production environments |
| `_later`/`_now` convention | Model triggers delivery; mailer is just the template layer |

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
    original_to = message.to
    message.to = ["staging-inbox@example.com"]
    message.cc = nil
    message.bcc = nil
    message.subject = "[STAGING] #{message.subject} (was: #{original_to.join(', ')})"
  end
end

# config/initializers/mail_interceptors.rb
if Rails.env.staging?
  ActionMailer::Base.register_interceptor(StagingInterceptor)
end
```

## Previews

```ruby
# Preview lives under the project's test directory:
#   test/mailers/previews/  or  spec/mailers/previews/
class OrderMailerPreview < ActionMailer::Preview
  def confirmation
    order = Order.first || Order.new(id: 1, number: "PREVIEW-001", created_at: Time.current)
    OrderMailer.confirmation(order)
  end

  def confirmation_with_discount
    order = Order.joins(:discount).first
    OrderMailer.confirmation(order)
  end
end
# Visit: http://localhost:3000/rails/mailers/order_mailer/confirmation
```

## Delivery Triggers

What triggers `deliver_later` depends on Axis A — see [delivery-trigger.native.md](delivery-trigger.native.md) or [delivery-trigger.extracted.md](delivery-trigger.extracted.md).

## Testing Mailers

Assert the same things regardless of framework: the email enqueues (or sends), the recipients and subject are correct, and the body includes the expected content. `ActionMailer::TestCase` / `assert_emails` and `have_enqueued_mail` are the core tools.

Write the test in the project's framework — read it from the `project-conventions` fingerprint (Testing category). Framework-specific patterns belong to `rails-testing-patterns`; do not infer the framework from architecture.

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
4. **Test file** in the project's framework (read it from the `project-conventions` fingerprint)
5. **Delivery configuration** notes if relevant
