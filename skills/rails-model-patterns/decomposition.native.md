# Model Decomposition — `native` axis

Domain logic lives in the model and its concerns. Callbacks are a normal, accepted tool.

## Callbacks

Use callbacks freely for simple, predictable side effects ("whenever X happens, do Y"). Keep each callback single-purpose — no multi-step orchestration inside a lifecycle hook.

## Concerns carry domain logic

Concerns are the primary tool for decomposing a model — not only behavior shared across models, but domain logic for a single model. Slice by trait or role: `Triageable`, `Postponable`, `Closeable`.

```ruby
# app/models/concerns/closeable.rb
module Closeable
  extend ActiveSupport::Concern
  included do
    has_one :closure, dependent: :destroy
    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def closed? = closure.present?
  def close!(by:, reason: nil) = create_closure!(creator: by, reason: reason)
  def reopen! = closure&.destroy!
end

# app/models/card.rb — composed from concerns
class Card < ApplicationRecord
  include Closeable
  include Triageable
  include Postponable
end
```

## POROs for operations

Plain Ruby objects for operations that don't fit a model. Not "service objects" — just objects, often in `app/models/`.

```ruby
# app/models/signup.rb
class Signup
  include ActiveModel::Model
  attr_accessor :name, :email, :password
  validates :name, :email, :password, presence: true

  def save
    return false unless valid?
    User.create!(name: name, email: email, password: password)
  end
end
```

## Decomposition fixes

- **Fat model (500+ lines)** → extract concerns sliced by trait; extract POROs for operations.
- **Callback doing complex orchestration** → keep the callback simple; move the multi-step workflow into an explicit model method that the controller calls.
