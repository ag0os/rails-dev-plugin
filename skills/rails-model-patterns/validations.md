# ActiveRecord Validation Patterns

Generate standard Rails validations (presence, uniqueness, format, length, numericality, inclusion, conditional) following Rails conventions. This document covers strategy decisions and non-obvious patterns.

## Core Rule: Always Pair with Database Constraints

Every critical validation must have a matching database constraint:

| Validation | DB Constraint |
|-----------|---------------|
| `presence: true` | `null: false` |
| `uniqueness: true` | `add_index ..., unique: true` |
| `numericality: { greater_than: 0 }` | `add_check_constraint` (PostgreSQL) |
| `inclusion: { in: [...] }` | `create_enum` or `add_check_constraint` |

Uniqueness validations are race-condition-prone without a unique index.

## Custom Validator Classes

Extract reusable validators to `app/validators/` when the same validation logic appears in multiple models:

```ruby
# app/validators/email_validator.rb
class EmailValidator < ActiveModel::EachValidator
  def validate_each(record, attribute, value)
    return if value.blank?
    unless value =~ URI::MailTo::EMAIL_REGEXP
      record.errors.add(attribute, options[:message] || "is not a valid email")
    end
    if options[:disposable] == false && disposable_email?(value)
      record.errors.add(attribute, "cannot be a disposable email")
    end
  end

  private

  def disposable_email?(email)
    domain = email.split('@').last
    DisposableDomains.include?(domain)
  end
end

# Usage: validates :email, email: { disposable: false }
```

## Validation Concerns for Cross-Cutting Patterns

```ruby
# app/models/concerns/sluggable.rb
module Sluggable
  extend ActiveSupport::Concern
  included do
    validates :slug, presence: true, uniqueness: true,
              format: { with: /\A[a-z0-9-]+\z/ }
    before_validation :generate_slug, on: :create
  end

  private

  def generate_slug
    self.slug ||= name&.parameterize
  end
end
```

## Validation Contexts

Use custom contexts sparingly — only when a model genuinely has different validity rules in different workflows:

```ruby
validates :terms_accepted, acceptance: true, on: :registration
user.save(context: :registration)
```

## Strict Validations

Use for programmer errors (not user input errors):

```ruby
validates :token, presence: true, strict: true
# Raises ActiveModel::StrictValidationFailed
```

## Skipping Validations

Use cautiously and document why:

- `save(validate: false)` — data migrations only
- `update_column` / `update_columns` — targeted fixes, skips callbacks too
- `insert_all` — bulk imports with pre-validated data

## Key Rules

- Combine related validations into one `validates` call per attribute
- Use `on: :create` / `on: :update` only when genuinely needed
- Prefer built-in validators over custom methods for standard checks
- Always add database constraints for critical validations — model validations alone are not sufficient for data integrity
- Use meaningful error messages, especially for user-facing validations
- Use i18n (`config/locales/`) for error messages in multi-locale apps
