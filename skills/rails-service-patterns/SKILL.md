---
name: rails-service-patterns
description: Analyzes and recommends Rails service object patterns for business logic extraction including command objects, result objects, form objects, query objects, and external API integration. Use when extracting logic from controllers/models, orchestrating multi-step workflows, or organizing app/services. NOT for simple CRUD, model validations, controller routing, or background job scheduling.
allowed-tools: Read, Grep, Glob
---

# Rails Service Object Patterns

Analyze and recommend patterns for extracting and organizing business logic in Rails applications.

## Quick Reference

| Pattern | Use When | Entry Point |
|---------|----------|-------------|
| Basic Service | Single operation with transaction | `CreateOrder.new(...).call` |
| Result Object | Caller needs success/failure + data | `Result.new(success?: true, data:)` |
| Form Object | Multi-model form submissions | `RegistrationForm.new(params).save` |
| Query Object | Complex reusable queries | `UserSearchQuery.new(scope).call` |
| Policy Object | Authorization decisions | `PostPolicy.new(user, post).update?` |

## Supporting Documentation

- [patterns.md](patterns.md) - Detailed service implementations and examples

## Core Principles

1. **Single responsibility**: One service, one operation (verb + noun naming)
2. **One public method**: Expose `call` or `perform` only
3. **Dependency injection**: Accept collaborators via constructor for testability
4. **Explicit return values**: Use Result objects, not exceptions for flow control
5. **Transaction boundaries**: Wrap multi-model changes in transactions

## Basic Service Pattern

```ruby
class CreateOrder
  def initialize(user:, cart_items:, payment_method:)
    @user = user
    @cart_items = cart_items
    @payment_method = payment_method
  end

  def call
    ActiveRecord::Base.transaction do
      order = @user.orders.create!(total: calculate_total, status: "pending")
      create_line_items(order)
      charge_payment(order)
      OrderMailer.confirmation(order).deliver_later
      order
    end
  end

  private

  def calculate_total = @cart_items.sum(&:price)
  def create_line_items(order) = # ...
  def charge_payment(order) = # ...
end
```

## Result Object Pattern

```ruby
class AuthenticateUser
  Result = Struct.new(:success?, :user, :error, keyword_init: true)

  def initialize(email:, password:)
    @email = email
    @password = password
  end

  def call
    user = User.find_by(email: @email)
    if user&.authenticate(@password)
      Result.new(success?: true, user: user)
    else
      Result.new(success?: false, error: "Invalid credentials")
    end
  end
end

# Controller usage
result = AuthenticateUser.new(email: params[:email], password: params[:password]).call
if result.success?
  sign_in(result.user)
else
  flash.now[:alert] = result.error
  render :new, status: :unprocessable_entity
end
```

## Dependency Injection

```ruby
class NotificationService
  def initialize(user:, mailer: UserMailer, sms: TwilioClient.new)
    @user = user
    @mailer = mailer
    @sms = sms
  end

  def call(message)
    @mailer.notification(@user, message).deliver_later
    @sms.send_sms(@user.phone, message) if @user.sms_enabled?
  end
end

# In tests: NotificationService.new(user: user, mailer: spy, sms: spy)
```

## External API Integration

```ruby
class WeatherService
  class ApiError < StandardError; end

  def initialize(api_key: Rails.application.credentials.weather_api_key, client: HTTParty)
    @api_key = api_key
    @client = client
  end

  def current_weather(city)
    response = @client.get("https://api.weather.com/current/#{city}", query: { api_key: @api_key })
    return Result.new(success?: true, data: parse(response)) if response.success?

    Result.new(success?: false, error: "API returned #{response.code}")
  rescue Net::OpenTimeout, Net::ReadTimeout => e
    Result.new(success?: false, error: "Timeout: #{e.message}")
  end
end
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| God service (100+ lines) | Does too much | Split into composable services |
| Raising exceptions for flow control | Expensive, hard to handle | Use Result objects |
| Service calling another service deeply | Hidden coupling, hard to debug | Orchestrate from controller or use a coordinator |
| No return value | Caller can't react to failures | Always return Result or meaningful value |
| Injecting ActiveRecord classes | Hard to test, tight coupling | Inject instances or use DI defaults |
| Class method services (`self.call`) | No instance state, limits DI | Use instance methods with constructor DI |
| Service modifying passed-in objects | Surprising side effects | Return new objects or be explicit |

## Output Format

When analyzing or creating services, provide:
1. **Service file** in `app/services/` with clear naming (VerbNoun)
2. **Result struct** if callers need success/failure status
3. **Controller integration** showing how to call and handle results
4. **Test outline** covering happy path, failure cases, and edge cases
5. **Error handling** strategy (Result objects vs exceptions)

## Error Handling

- Use `ActiveRecord::Base.transaction` with `save!` / `create!` to auto-rollback on failure
- Rescue specific exceptions, never bare `rescue`
- Return Result objects for expected failures (validation, auth)
- Raise exceptions for unexpected failures (network, system)
- Log errors with context: `Rails.logger.error("[CreateOrder] Payment failed: #{e.message}")`
- Define custom exception classes per domain: `class PaymentError < StandardError; end`
