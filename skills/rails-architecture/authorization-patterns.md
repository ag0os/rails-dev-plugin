# Authorization and Multi-Tenancy Patterns

Reference guide for authorization patterns in this Rails application.

## Overview

This application uses:
- **Multi-tenancy** via `Account` and `AccountUser` models
- **Pundit** for declarative authorization
- **Three authorization layers**: System, Account, Resource

## Authorization Layers

### Layer 1: System Level (user.admin)
- Access ALL resources across ALL accounts
- Used for platform maintenance, migrations
- Check: `account_user.user.admin?`

### Layer 2: Account Level (account_user roles)
- **Admin**: Manage account settings, full CRUD
- **Member**: View and create resources
- **Producer**: Domain-specific permissions
- Check: `account_user.admin?`, `account_user.member?`

### Layer 3: Resource Level (account_person)
- **Read**: View only
- **Write**: View and edit
- **Admin**: Full control including access management
- Check: `person.accessible_by_account?(account)`

## Common Patterns

### Pattern 1: Policy with System Admin Override

```ruby
class Insurance::PolicyPolicy < ApplicationPolicy
  def update?
    return true if account_user.user.admin?  # System admin
    return true if account_user.admin?       # Account admin

    # Member can edit if has write access
    record.person&.can_be_edited_by_account?(account_user.account)
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

### Pattern 2: Controller Authorization

```ruby
class Insurance::PoliciesController < ApplicationController
  before_action :set_policy, only: [:show, :edit, :update]

  def index
    @policies = policy_scope(Insurance::Policy)  # Scopes to account
    authorize Insurance::Policy                   # Authorizes action
  end

  def show
    # @policy loaded and authorized in before_action
  end

  private

  def set_policy
    @policy = Insurance::Policy.find(params[:id])
    authorize @policy
  end
end
```

### Pattern 3: Model Scoping

```ruby
class Crm::Person < ApplicationRecord
  has_many :account_people
  has_many :accounts, through: :account_people

  scope :accessible_by_account, ->(account) {
    joins(:account_people).where(account_people: { account: account })
  }

  def accessible_by_account?(account)
    accounts.include?(account)
  end

  def can_be_edited_by_account?(account)
    account_person = account_people.find_by(account: account)
    account_person&.write? || account_person&.admin?
  end
end
```

## Current Context (Request-Scoped)

```ruby
# Available throughout the request
Current.user          # => #<User id: 1>
Current.account       # => #<Account id: 5>
Current.account_user  # => #<AccountUser roles: {admin: true, member: true}>

# Usage in policies
def update?
  return true if account_user.user.admin?
  account_user.admin?
end
```

## Security Checklist

- ✅ Always use `policy_scope` for collections
- ✅ Always call `authorize` for actions
- ✅ Check system admin first in policies
- ✅ Never use `Model.all` in controllers
- ✅ Use `AccountUser` (not `User`) as `pundit_user`
- ✅ Test authorization in controller tests

## Common Mistakes

### ❌ Forgetting to authorize
```ruby
def show
  @policy = Insurance::Policy.find(params[:id])
  # Missing: authorize @policy
end
```

### ❌ Not scoping collections
```ruby
def index
  @policies = Insurance::Policy.all  # Shows ALL accounts!
end
```

### ❌ Skipping system admin check
```ruby
def update?
  account_user.admin?  # Missing system admin check!
end
```

## Reference

For complete details, see `docs/architecture/authorization-and-access-control.md`

Key models:
- `User` - Individual users (has `admin` flag)
- `Account` - Tenants (organizations or personal)
- `AccountUser` - Join table with roles
- `AccountPerson` - Resource-level access grants
