---
name: ruby-object-design
description: Automatically invoked when making decisions about Ruby code structure and organization. Triggers on "class or module", "should this be a class", "struct vs class", "PORO", "data object", "design pattern", "class vs module", "when to use class", "module vs class", "stateless class", "value object", "data container", "object factory", "extend self", "singleton class". Provides guidance on choosing the right Ruby construct (class, module, Struct, Data, Hash). NOT for code smell identification or refactoring (use ruby-refactoring) or Rails-specific framework patterns.
allowed-tools: Read, Grep, Glob
---

# Ruby Object Design Expert

**Ruby is object-oriented, not class-oriented.** The `class` keyword should be reserved for specific use cases, not used as a default container for code.

## The Object Factory Rule

**Only use `class` if you are creating an object factory** -- a template for generating multiple objects that encapsulate internal state with behaviors operating on that state.

If your code doesn't create multiple instances with distinct state, a class is the wrong construct.

## Decision Tree

```
Do you need multiple instances with encapsulated state?
|-- YES: Does the object have both state AND behavior?
|   |-- YES -> Class
|   +-- NO (just data) -> Struct or Data
+-- NO: Is this a collection of related functions?
    |-- YES -> Module with `extend self`
    +-- NO: Is this a one-off transformation?
        +-- YES -> standalone method or lambda
```

## Red Flags: When NOT to Use a Class

### 1. Stateless Utility Buckets

If your class has no instance variables, it's a module pretending to be a class.

```ruby
# Bad: Class with no state
class StringUtils
  def self.titleize(string) = string.split.map(&:capitalize).join(' ')
end

# Good: Module with extend self
module StringUtils
  extend self
  def titleize(string) = string.split.map(&:capitalize).join(' ')
end
```

**Why module:** Clearer intent, can be `include`d, no misleading `.new` method.

### 2. Single-Method "Service" Classes

Classes with only `initialize` + `call` are often functions in disguise. Ask:
- Will multiple instances exist simultaneously?
- Does the object maintain state between method calls?
- Is `initialize` + `call` adding clarity or ceremony?

```ruby
# Questionable: function in a class costume
class CalculateDiscount
  def initialize(order) = @order = order
  def call = @order.subtotal * discount_rate
  private
  def discount_rate = @order.customer.premium? ? 0.1 : 0.05
end

# Alternative: Module function
module Discounts
  extend self
  def calculate(order) = order.subtotal * discount_rate(order.customer)
  private
  def discount_rate(customer) = customer.premium? ? 0.1 : 0.05
end
```

**Exception:** Service classes ARE appropriate when the Rails project follows service-oriented patterns consistently. Don't fight the codebase convention.

### 3. Classes Named After Design Patterns

If your class is named `Factory`, `Builder`, `Decorator`, `Adapter`, or `AbstractBase`, reconsider. Most GoF patterns were workarounds for C++/Java limitations and are unnecessary in Ruby.

See [class-vs-module.md](class-vs-module.md) for Ruby-native alternatives.

### 4. Objects Invalid After Initialization

If an object requires calling setters before it functions, fix the constructor:

```ruby
# Bad: requires setup ceremony after .new
report = ReportGenerator.new
report.set_data(data)      # Must call before generate!
report.generate

# Good: valid at birth
ReportGenerator.new(data: data).generate
```

## Data (Ruby 3.2+) for Immutable Value Objects

Prefer `Data.define` over classes for immutable value objects. It provides `==`, `hash`, `with`, and keyword-only `new` out of the box.

```ruby
Point = Data.define(:x, :y) do
  def distance_from_origin = Math.sqrt(x**2 + y**2)
  def translate(dx, dy) = with(x: x + dx, y: y + dy)  # returns new instance
end
```

**Pre-3.2 fallback:** Use frozen Struct with `keyword_init: true`.

See [data-structures.md](data-structures.md) for the full graduation path: Hash -> Struct -> Data -> Class.

## Decision Matrix

| Scenario | Use | Why |
|----------|-----|-----|
| Multiple instances with state + behavior | Class | True object factory |
| Stateless utility methods | Module with `extend self` | No state to encapsulate |
| Simple data container | Struct or Data | Avoids boilerplate |
| Immutable value object | Data (3.2+) or frozen Struct | Built-in immutability |
| Ad-hoc/temporary data | Hash | Simplest solution |
| Named after a design pattern | Rethink design | Patterns often unnecessary in Ruby |
| Invalid after `.new` without setup | Not a class | Objects must be valid at birth |

## Context Awareness

Before applying these principles, check the existing codebase:

1. **Check Ruby version**: `Data.define` requires 3.2+
2. **Check conventions**: If the project uses service objects consistently, follow suit
3. **Grep for patterns**: `grep -r "class.*Service" app/` to understand the project's style

## Related Documentation

- [class-vs-module.md](class-vs-module.md) - GoF pattern replacements with Ruby-native alternatives
- [data-structures.md](data-structures.md) - Struct, Data, Hash graduation path and value object patterns

## Output Format

When providing object design recommendations:
1. **Current State Analysis** -- what construct is used and why it may be suboptimal
2. **Recommended Construct** -- with rationale tied to the Object Factory Rule
3. **Before/After Code** -- showing the improvement
4. **Caveats** -- team conventions, Ruby version requirements
