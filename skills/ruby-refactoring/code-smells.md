# Code Smells: Rails-Specific Detection

This file covers detection heuristics specific to Rails apps that go beyond textbook definitions. For standard smell definitions, use established knowledge from Fowler's refactoring catalog and Ruby Science.

## Feature Envy in ActiveRecord Models

Standard Feature Envy is straightforward. In Rails, watch for these specific patterns:

**Association chain access** -- a method reaches through 2+ associations:
```ruby
# Smell: method lives on Order but mostly touches Customer and Address
def shipping_label
  "#{customer.name}\n#{customer.address.street}\n#{customer.address.city}, #{customer.address.state} #{customer.address.zip}"
end
```
Fix: push `shipping_label` to Address or use `delegate`. The method belongs where the data lives.

**Scope envy** -- a model defines scopes that filter on associated model columns:
```ruby
# Smell: Policy scope filtering on Person attributes
scope :for_senior_customers, -> { joins(:person).where("people.age >= ?", 65) }
```
Fix: define scope on Person (`Person.seniors`) and query through it, or accept this as a pragmatic join scope but document the coupling.

## Shotgun Surgery Across Rails Layers

This is the most costly smell in Rails apps and the hardest to detect mechanically.

**Detection heuristic**: When adding a new field or business rule, count the files touched. If a single concept change touches 4+ of these layers, you have shotgun surgery:
- Migration
- Model (validation, callback, scope)
- Controller (strong params, assignment)
- View/serializer (display)
- Test files (model spec, controller spec, system spec)
- Form object or decorator

**Common causes**:
- Business logic scattered between controller and model
- Display logic in both helpers and views
- Authorization checks duplicated in controller and view

**Fix**: consolidate the business rule into one object. Which object depends on stack profile:
- **Omakase**: model method or concern
- **Service-oriented**: service object or form object
- **API-first**: form object or policy object

## God Model Detection

A model is becoming a god object when it exhibits 3+ of:
- More than 10 `has_many`/`has_one` associations
- More than 5 callbacks (before/after)
- More than 200 lines
- Concerns count exceeds 5
- Methods that don't use `self` attributes (they belong elsewhere)

**Triage approach**: Don't refactor the entire god model at once. Identify the most actively-edited axis of change (use `git log --follow` on the file) and extract that concern first.

## Primitive Obsession with Status Fields

Rails apps frequently use string columns for status:
```ruby
# Smell: string status with scattered conditionals
if policy.status == "active" && policy.payment_status == "current"
```

Prefer Rails enums with scopes, or extract a state machine when transitions have business rules.

## Lazy Concern (Rails-Specific Lazy Class)

A concern that wraps a single method or a single scope is noise, not reuse. Inline it unless you have evidence it will be used in 2+ models.

## Summary: When to Escalate Smell Severity

Upgrade a smell's priority when:
- The file appears in >30% of recent PRs (`git log --oneline --diff-filter=M`)
- Multiple developers report confusion about where logic lives
- Test setup for the class requires >20 lines of factory setup
- The class has a Flog score above 100 or ABC score above 30
