# Data Structures: When to Graduate

## Graduation Path: Hash -> Struct -> Data -> Class

### Hash: Ad-hoc, temporary data

Use when the structure is one-off or throwaway. **Graduate to Struct when** you access the same keys repeatedly, pass the hash to multiple methods, or want to add behavior.

### Struct: Named data with optional behavior

```ruby
# Always use keyword_init for clarity
Point = Struct.new(:x, :y, keyword_init: true)
point = Point.new(x: 10, y: 20)

# Add behavior in the block
Point = Struct.new(:x, :y, keyword_init: true) do
  def distance_from_origin = Math.sqrt(x**2 + y**2)
end
```

### Data (Ruby 3.2+): Immutable value objects

Preferred for value objects. Provides `==` by value, `hash`, `with` for modified copies, and keyword-only initialization by default.

```ruby
Point = Data.define(:x, :y) do
  def distance_from_origin = Math.sqrt(x**2 + y**2)
  def translate(dx, dy) = with(x: x + dx, y: y + dy)
end

point = Point.new(x: 3, y: 4)
moved = point.translate(1, 1)  # => Point(x: 4, y: 5)
point                          # => Point(x: 3, y: 4) — unchanged
```

**Key difference from Struct:** No setters, no positional args, built-in `with` method.

### Pre-3.2 Immutability: Frozen Struct

```ruby
Point = Struct.new(:x, :y, keyword_init: true) do
  def initialize(*) = (super; freeze)
  def translate(dx, dy) = Point.new(x: x + dx, y: y + dy)
end
```

### Class: Only when you need full control

Graduate to class only when you need: private state, complex initialization logic, lifecycle management, or validation in the constructor.

## Common Value Object Patterns

### Money (use Data for 3.2+)

```ruby
Money = Data.define(:amount, :currency) do
  def +(other)
    raise "Currency mismatch" unless currency == other.currency
    with(amount: amount + other.amount)
  end

  def to_s = "#{currency} #{amount}"
end
```

### Result Object

```ruby
Result = Data.define(:success, :value, :error) do
  def self.success(value) = new(success: true, value: value, error: nil)
  def self.failure(error) = new(success: false, value: nil, error: error)
  def success? = success
  def failure? = !success
end
```

## Quick Decision

```
Temporary/one-off data? -> Hash
Named structure with accessors? -> Struct (mutable) or Data (immutable, 3.2+)
Complex initialization, private state, lifecycle? -> Class
```

Always check `.ruby-version` or `Gemfile` for Ruby version before recommending `Data.define`.
