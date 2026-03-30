# Refactoring Patterns: Selection Guidance

For standard pattern mechanics (Extract Method, Extract Class, etc.), apply established Fowler/Ruby Science patterns directly. This file provides guidance for **ambiguous cases** where the right pattern isn't obvious.

## When Extract Method vs. Replace Method with Method Object

Use **Extract Method** by default. Escalate to **Method Object** only when:
- The method has 4+ local variables that interact with each other
- Extracting methods would require passing 3+ parameters between them
- The algorithm is complex enough to warrant its own test suite

In Rails, Method Object often becomes a service object or calculator class (e.g., `PremiumCalculator`, `InvoiceTotalizer`).

## When Extract Class vs. Extract Concern

| Criterion | Extract Class | Extract Concern |
|-----------|--------------|-----------------|
| Shared across models? | No -- unique to one model | Yes -- 2+ models need it |
| Has its own state/data? | Yes -- deserves its own table | No -- operates on host's attributes |
| Testable in isolation? | Should be | May depend on host class |
| Profile preference | Service-oriented, API-first | Omakase |

**Anti-pattern**: extracting a concern used by only one model. This is just moving code without reducing complexity.

## When Service Object vs. Model Method

Extract to a service object when the operation:
- Touches 2+ models in a transaction
- Calls external APIs
- Has side effects (email, webhooks, job enqueue)
- Requires its own failure/success handling

Keep as a model method when:
- It only reads/writes the model's own attributes
- It's a pure calculation on the model's data
- It's used in scopes or validations

## Replace Conditional with Polymorphism: STI vs. Strategy

Use **STI** when:
- Subtypes are persisted and queried differently
- Rails conventions support it (single `type` column)
- Subtypes share most columns

Use **Strategy pattern** (plain Ruby classes) when:
- Behavior varies but data structure is identical
- You don't want to pollute the DB schema
- Subtypes are transient (not persisted)

```ruby
# Strategy pattern -- when STI is overkill
class Policy < ApplicationRecord
  def premium_calculator
    "#{insurance_type.camelize}PremiumCalculator".constantize.new(self)
  end
end
```

## Introduce Parameter Object: When to Use Struct vs. Class

- Use `Data.define` (Ruby 3.2+) for immutable parameter objects
- Use `Struct` for simple cases where mutability is acceptable
- Use a full class when the parameter object needs validation, coercion, or derived attributes

## Refactoring Safety Checklist

Before starting any refactoring:
1. Confirm test coverage exists for the code path (run coverage tool if uncertain)
2. Make one small change, run tests, commit
3. Repeat -- never batch multiple refactoring patterns into one commit
4. If tests don't exist, write characterization tests BEFORE refactoring
5. Keep each PR focused on one smell/pattern to ease review
