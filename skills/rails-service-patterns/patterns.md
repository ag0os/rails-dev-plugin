# Service Object Patterns — Detailed Reference

## Enhanced Result with Monad-Like Chaining

The simple `Struct` result works for most cases. When you need chainable callbacks:

```ruby
class ServiceResult
  attr_reader :value, :error

  def self.success(value) = new(value: value, success: true)
  def self.failure(error) = new(error: error, success: false)

  def initialize(value: nil, error: nil, success:)
    @value = value
    @error = error
    @success = success
  end

  def success? = @success
  def failure? = !@success

  def on_success
    yield(value) if success?
    self
  end

  def on_failure
    yield(error) if failure?
    self
  end
end

# Usage with chaining
ProcessPayment.new(order).call
  .on_success { |payment| redirect_to payment }
  .on_failure { |error| render :new, alert: error }
```

## Form Object Pattern

Use `ActiveModel::Model` for complex forms spanning multiple models. The form replaces the controller's role of coordinating creation across models.

```ruby
class RegistrationForm
  include ActiveModel::Model
  include ActiveModel::Validations

  attr_accessor :email, :password, :password_confirmation,
                :company_name, :company_size

  validates :email, presence: true, format: { with: URI::MailTo::EMAIL_REGEXP }
  validates :password, presence: true, length: { minimum: 8 }, confirmation: true
  validates :company_name, presence: true

  def save
    return false unless valid?

    ActiveRecord::Base.transaction do
      user = User.create!(email: email, password: password)
      Company.create!(name: company_name, size: company_size, owner: user)
      true
    end
  rescue ActiveRecord::RecordInvalid => e
    errors.add(:base, e.message)
    false
  end
end

# Controller stays thin
def create
  @form = RegistrationForm.new(registration_params)
  if @form.save
    redirect_to dashboard_path
  else
    render :new
  end
end
```

## Composable Query Objects

For complex queries that need to be reused and composed. Follow standard Rails conventions for basic scopes — only extract a query object when the query logic is reusable across controllers or involves complex joins/subqueries.

```ruby
class PolicySearch
  def initialize(relation = Policy.all)
    @relation = relation
  end

  def by_status(status)
    @relation = @relation.where(status: status)
    self
  end

  def expiring_within(days)
    @relation = @relation.where("expiry_date < ?", days.from_now)
    self
  end

  def for_agent(agent)
    @relation = @relation.where(agent: agent)
    self
  end

  def results = @relation
end

# Chainable usage
PolicySearch.new
  .by_status("active")
  .expiring_within(30.days)
  .for_agent(current_agent)
  .results
```

## Directory Organization

Group services by domain when you have more than ~10 service files:

```
app/services/
  orders/
    create_order.rb
    cancel_order.rb
  payments/
    process_payment.rb
    refund_payment.rb
  notifications/
    send_welcome.rb
```

Keep a flat `app/services/` structure until the domain grouping becomes necessary. Premature namespacing adds indirection without value.

## Error Handling Strategy

| Error Type | Handling | Example |
|-----------|---------|---------|
| Expected failure (validation, auth) | Return `Result.failure(...)` | Invalid credentials, payment declined |
| Unexpected failure (network, system) | Raise exception, let caller rescue | `Net::OpenTimeout`, database down |
| Domain rule violation | Return `Result.failure(...)` | Insufficient funds, expired subscription |

**Key rule:** If the caller needs to branch on the outcome, use a Result. If the caller can't meaningfully recover, raise an exception.

```ruby
class ProcessPayment
  def call
    # Expected failure -> Result
    return failure("Amount must be positive") if @amount <= 0

    # Unexpected failure -> let it raise (or rescue at top level)
    response = PaymentGateway.charge(@amount, @token)

    if response.success?
      success(response.charge)
    else
      failure(response.error_message)  # Expected: card declined, etc.
    end
  end
end
```
