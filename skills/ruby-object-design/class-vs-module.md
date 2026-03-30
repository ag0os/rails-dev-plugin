# Class vs Module — GoF Pattern Replacements

Ruby has built-in features that make most Gang of Four patterns unnecessary. When you see a class named after a pattern, consider these Ruby-native alternatives.

## Factory Pattern -> Hash Lookup or Direct Instantiation

```ruby
# Unnecessary factory class
class NotificationFactory
  def self.create(type, message)
    case type
    when :email then EmailNotification.new(message)
    when :sms then SmsNotification.new(message)
    end
  end
end

# Ruby way: hash lookup for dynamic dispatch
NOTIFICATION_TYPES = {
  email: EmailNotification,
  sms: SmsNotification,
  push: PushNotification
}.freeze

def create_notification(type, message)
  NOTIFICATION_TYPES.fetch(type).new(message)
end

# Or just: EmailNotification.new(message) — skip the indirection
```

## Decorator Pattern -> Module Prepend

```ruby
# Java-style decorator class
class LoggingDecorator
  def initialize(service) = @service = service
  def call(args)
    Rails.logger.info("Calling with #{args}")
    @service.call(args)
  end
end

# Ruby way: prepend a module
module Logging
  def call(args)
    Rails.logger.info("Calling with #{args}")
    super.tap { |result| Rails.logger.info("Returned #{result}") }
  end
end

class MyService
  prepend Logging
end
```

## Strategy Pattern -> Procs/Lambdas

```ruby
# Verbose strategy hierarchy
class PricingStrategy
  def calculate(order); raise NotImplementedError; end
end
class PremiumPricing < PricingStrategy
  def calculate(order) = order.subtotal * 0.9
end

# Ruby way: lambdas
PRICING = {
  regular: ->(order) { order.subtotal },
  premium: ->(order) { order.subtotal * 0.9 }
}.freeze

def calculate_price(order, tier) = PRICING.fetch(tier).call(order)
```

## Abstract Base Class -> Module Mixin

```ruby
# Bad: Inheritance for method injection
class BaseProcessor
  def process = (validate; execute; notify)
  def execute = raise NotImplementedError
end

# Good: Composition with modules
module Processable
  def process = (validate; execute; notify)
  private
  def validate = nil
  def notify = nil
end

class OrderProcessor
  include Processable
  def execute = # implementation
end
```

## When Service Classes ARE Appropriate

A class with `initialize` + `call` is justified when:
- It maintains state across multiple operations (e.g., `BatchEmailSender` tracking sent count and failures)
- The project consistently uses this pattern (don't fight conventions)
- Constructor DI is needed for testability with multiple collaborators

```ruby
# Justified: state across operations
class BatchEmailSender
  def initialize(users)
    @users = users
    @sent_count = 0
    @failed = []
  end

  def send_all
    @users.each { |user| send_to(user) }
    self
  end

  def report = { sent: @sent_count, failed: @failed }
end
```
