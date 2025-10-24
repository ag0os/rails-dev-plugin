---
name: Rails Architecture
description: Provides architectural guidance for Rails applications including design patterns, service layer architecture, model organization, multi-tenancy patterns, and decision frameworks. Use when making architectural decisions, designing new features, evaluating refactoring approaches, or questions about Rails patterns and conventions in this project.
allowed-tools: Read, Grep, Glob
---

# Rails Architecture Guidance

## When to Use This Skill

- Designing new feature modules or domain models
- Deciding on service object patterns vs model methods
- Evaluating model vs service layer placement decisions
- Choosing between architectural approaches
- Refactoring complex controller or model logic
- Understanding multi-tenancy and authorization patterns
- Making decisions about concerns, modules, and code organization

## Core Architectural Principles

### Evidence-Based Design

**Start simple, design when needed**: Don't anticipate requirements. Wait for concrete evidence before adding complexity.

**DRY applies to knowledge, not code**: If two code blocks look identical but represent different concepts, don't combine them. The coffee recipe and tea recipe are different knowledge even if they share steps.

**Two instances aren't a pattern**: Wait for three or more occurrences before abstracting. You'll have better information about what the abstraction should be.

**Ask "How easy is it to change?"**: This is the primary metric for evaluating any design decision.

### When to Refactor

Refactor when:
- Code becomes difficult to understand at a glance
- Clear evidence that change is harder than it should be
- Pattern repeats 3+ times
- NOT when you see similar-looking code
- NOT based on imagined future requirements

## Layer Responsibilities

### 1. Model Layer

**Purpose**: Domain logic and data relationships

**Good model code**:
- Domain validations and business rules
- Associations and relationships
- Data integrity constraints
- Scopes for common queries
- Simple calculations based on own attributes

**Extract to service when**:
- Logic spans multiple models
- External API calls required
- Complex orchestration needed
- Heavy side effects (emails, jobs, etc.)

```ruby
# ✅ Good: Model handles its own domain logic
class Insurance::Policy < ApplicationRecord
  belongs_to :person, class_name: "Crm::Person"
  belongs_to :company, class_name: "Insurance::Company"

  validates :policy_number, presence: true, uniqueness: true
  validates :effective_date, presence: true

  scope :active, -> { where(status: 'active') }
  scope :expiring_soon, -> { where('expiry_date BETWEEN ? AND ?', Date.today, 30.days.from_now) }

  def premium_per_month
    total_premium / 12.0
  end

  def renewable?
    active? && expiry_date > Date.today
  end
end

# ❌ Bad: Model doing too much orchestration
class Insurance::Policy < ApplicationRecord
  def renew!(new_expiry_date)
    transaction do
      # Orchestrating across models - should be a service
      old_policy = dup
      old_policy.update!(status: 'renewed')

      update!(
        effective_date: Date.today,
        expiry_date: new_expiry_date,
        previous_policy: old_policy
      )

      # Side effects - should be in service
      PolicyMailer.renewal_confirmation(self).deliver_later
      create_renewal_invoice
      notify_producers
    end
  end
end
```

### 2. Service Layer

**Purpose**: Complex business operations that coordinate multiple models

**Use service objects for**:
- Operations spanning 2+ models
- External API integrations
- Complex validation/error handling
- Multi-step workflows with rollback needs

**Service object pattern**:

```ruby
# app/services/insurance/policy_renewal_service.rb
module Insurance
  class PolicyRenewalService
    def initialize(policy, new_expiry_date, renewed_by:)
      @policy = policy
      @new_expiry_date = new_expiry_date
      @renewed_by = renewed_by
    end

    def call
      return failure("Policy not renewable") unless @policy.renewable?

      ApplicationRecord.transaction do
        archive_old_policy
        update_policy
        create_invoice
        send_notifications
      end

      success(@policy)
    rescue StandardError => e
      failure(e.message)
    end

    private

    def archive_old_policy
      # Archive logic
    end

    def update_policy
      @policy.update!(
        effective_date: Date.today,
        expiry_date: @new_expiry_date
      )
    end

    def create_invoice
      # Invoice creation
    end

    def send_notifications
      PolicyMailer.renewal_confirmation(@policy).deliver_later
    end

    def success(result)
      { success: true, policy: result }
    end

    def failure(error)
      { success: false, error: error }
    end
  end
end

# Usage in controller:
result = Insurance::PolicyRenewalService.new(
  @policy,
  params[:new_expiry_date],
  renewed_by: current_user
).call

if result[:success]
  redirect_to result[:policy]
else
  flash[:error] = result[:error]
  render :edit
end
```

### 3. Controller Layer

**Purpose**: HTTP interface and request handling

**Thin controllers principles**:
- One action = one responsibility
- Delegate to models or services
- Handle params and rendering
- Authorization via Pundit
- Use before_actions for setup only

```ruby
# ✅ Good: Thin controller
class Insurance::PoliciesController < ApplicationController
  before_action :set_policy, only: [:show, :edit, :update]

  def index
    @policies = policy_scope(Insurance::Policy)
    authorize Insurance::Policy
    @pagy, @policies = pagy(@policies)
  end

  def show
    # Authorization done in before_action
  end

  def update
    if @policy.update(policy_params)
      redirect_to @policy, notice: "Policy updated"
    else
      render :edit
    end
  end

  def renew
    result = Insurance::PolicyRenewalService.new(
      @policy,
      params[:new_expiry_date],
      renewed_by: current_user
    ).call

    if result[:success]
      redirect_to result[:policy]
    else
      flash.now[:error] = result[:error]
      render :edit
    end
  end

  private

  def set_policy
    @policy = Insurance::Policy.find(params[:id])
    authorize @policy
  end

  def policy_params
    params.require(:insurance_policy).permit(policy(@policy).permitted_attributes)
  end
end

# ❌ Bad: Fat controller with business logic
class Insurance::PoliciesController < ApplicationController
  def renew
    @policy = Insurance::Policy.find(params[:id])

    # Business logic in controller - should be service
    old_policy = @policy.dup
    old_policy.status = 'renewed'
    old_policy.save!

    @policy.effective_date = Date.today
    @policy.expiry_date = params[:new_expiry_date]
    @policy.previous_policy = old_policy

    if @policy.save
      PolicyMailer.renewal_confirmation(@policy).deliver_later
      Invoice.create!(policy: @policy, amount: @policy.total_premium)
      redirect_to @policy
    else
      render :edit
    end
  end
end
```

## Multi-Tenancy Architecture

This application uses **account-based multi-tenancy**. See [authorization-patterns.md](authorization-patterns.md) for complete details.

### Key Concepts

**Three authorization layers**:
1. **System Level** - `user.admin?` - Access everything across all accounts
2. **Account Level** - `account_user.admin?` - Manage current account
3. **Resource Level** - `account_person.access_level` - Fine-grained permissions

**Current context** (request-scoped):
- `Current.user` - Logged-in user
- `Current.account` - Selected account
- `Current.account_user` - Join record with roles

**Pundit integration**:
- Uses `AccountUser` (not `User`) as `pundit_user`
- Provides both user and account context to policies
- Scoping via `policy_scope(Model)`

### Multi-Tenant Pattern Examples

```ruby
# ✅ Good: Properly scoped to account
class Insurance::PoliciesController < ApplicationController
  def index
    # Scopes to current account automatically
    @policies = policy_scope(Insurance::Policy)
    authorize Insurance::Policy
  end
end

# Policy with proper scoping
class Insurance::PolicyPolicy < ApplicationPolicy
  class Scope < Scope
    def resolve
      return scope.all if account_user.user.admin?

      # Regular users see policies via account_people
      scope.joins(person: :account_people)
        .where(account_people: { account_id: account_user.account_id })
        .distinct
    end
  end
end

# ❌ Bad: No account scoping - data leak!
def index
  @policies = Insurance::Policy.all  # Shows ALL accounts' data!
end
```

## Decision Frameworks

### When to Extract a Service Object

Extract when you have **3+ of these indicators**:

- [ ] Business logic spans multiple models
- [ ] Operation requires external API calls
- [ ] Complex validation or error handling
- [ ] Operation needs to be tested in isolation
- [ ] Logic doesn't naturally belong in any model
- [ ] Heavy side effects (emails, jobs, notifications)
- [ ] Multi-step workflow requiring transaction

**Examples of service-worthy operations**:
- Policy renewal (updates policy, creates invoice, sends emails)
- Data import from external system
- Payment processing with external gateway
- Complex report generation

**Keep in model when**:
- Logic only uses model's own attributes
- Simple calculation or formatting
- Standard validation
- Single responsibility, single model

### When to Use Concerns

**Use concerns for**:
- Shared behavior across multiple models
- Cross-cutting functionality (e.g., Archivable, Searchable)
- Clear, focused responsibility

```ruby
# ✅ Good: Focused concern
module Archivable
  extend ActiveSupport::Concern

  included do
    scope :archived, -> { where.not(archived_at: nil) }
    scope :active, -> { where(archived_at: nil) }
  end

  def archive!
    update!(archived_at: Time.current)
  end

  def archived?
    archived_at.present?
  end
end

# Usage:
class Insurance::Policy < ApplicationRecord
  include Archivable
end
```

**Avoid concerns when**:
- Logic used in only one place
- Abstraction isn't clear
- Creates hidden dependencies
- Makes code harder to trace

### When to Add Validation

**Add validation at model level when**:
- Data integrity requirement
- Business rule about the data itself
- Should fail in any context

**Add validation in service when**:
- Contextual business rules
- Depends on user action or workflow
- Involves multiple models

```ruby
# Model validation - always required
class Insurance::Policy < ApplicationRecord
  validates :policy_number, presence: true, uniqueness: true
  validates :effective_date, presence: true
end

# Service validation - contextual
class PolicyRenewalService
  def call
    return failure("Cannot renew expired policy") if @policy.expired?
    return failure("Renewal date must be after current expiry") if @new_date <= @policy.expiry_date
    # ... proceed with renewal
  end
end
```

## Common Patterns

### Pattern: Policy Scoping (Multi-Tenant)

```ruby
class Insurance::PolicyPolicy < ApplicationPolicy
  def show?
    return true if account_user.user.admin?  # System admin
    record.person&.accessible_by_account?(account_user.account)
  end

  class Scope < Scope
    def resolve
      return scope.all if account_user.user.admin?

      scope.joins(person: :account_people)
        .where(account_people: { account_id: account_user.account_id })
        .distinct
    end
  end
end
```

### Pattern: Service Object with Result Object

```ruby
class MyService
  def call
    # Do work
    success(result)
  rescue => e
    failure(e.message)
  end

  private

  def success(data)
    { success: true, data: data }
  end

  def failure(error)
    { success: false, error: error }
  end
end
```

### Pattern: Form Object (for complex forms)

```ruby
class PolicyRenewalForm
  include ActiveModel::Model

  attr_accessor :policy, :new_expiry_date, :adjustment_reason

  validates :new_expiry_date, presence: true
  validate :expiry_date_is_valid

  def save
    return false unless valid?

    PolicyRenewalService.new(
      policy,
      new_expiry_date,
      adjustment_reason: adjustment_reason
    ).call
  end

  private

  def expiry_date_is_valid
    if new_expiry_date && new_expiry_date <= policy.expiry_date
      errors.add(:new_expiry_date, "must be after current expiry date")
    end
  end
end
```

## Related Documentation

- [authorization-patterns.md](authorization-patterns.md) - Multi-tenancy and Pundit patterns
- [service-objects.md](service-objects.md) - Service object patterns and examples
- [testing-patterns.md](testing-patterns.md) - Testing strategies for each layer

## Quick Reference

**Model**: Domain logic, validations, associations, simple calculations
**Service**: Multi-model operations, external APIs, complex workflows
**Controller**: HTTP handling, authorization, delegation
**Policy**: Authorization rules and scoping
**Concern**: Shared behavior across models (use sparingly)

**Always remember**: Evidence-based design. Start simple. Extract when you have evidence, not speculation.
