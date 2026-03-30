# Value Objects in Rails

Generate standard Struct and Data value objects following Ruby conventions. This document covers decision guidance and Rails integration patterns.

## When to Use What

| Start With | Upgrade When | Upgrade To |
|------------|--------------|------------|
| Hash | Same keys appear 3+ times | Struct |
| Struct | Need immutability | Data (Ruby 3.2+) |
| Struct | Need validations/forms | ActiveModel::Model |
| Data | Need mutability | Struct |
| Value Object | Need persistence | ApplicationRecord |

**Rule of thumb:** Don't create value objects for single attributes. Use them for multi-attribute domain concepts (Money, Address, DateRange, Coordinate).

## Data vs Struct Decision

- **Ruby 3.2+**: Use `Data.define` — immutable by default, keyword args, `#with` for copies
- **Ruby < 3.2**: Use `Struct.new` with `freeze` in `initialize`

```ruby
# Ruby 3.2+ (preferred)
Coordinate = Data.define(:lat, :lng)

# Ruby < 3.2 (fallback)
Coordinate = Struct.new(:lat, :lng) do
  def initialize(lat:, lng:)
    super(lat, lng)
    freeze
  end
end
```

## Enum-like Struct Pattern

For fixed sets of domain values (colors, priorities, statuses):

```ruby
Color = Struct.new(:name, :value) do
  class << self
    def for_value(value)
      COLORS.find { |it| it.value == value }
    end
  end

  def to_s = value

  COLORS = [
    new("Blue", "var(--color-card-default)"),
    new("Green", "var(--color-card-2)"),
    new("Red", "var(--color-card-3)"),
  ].freeze
end
```

## Rails Database Integration

### Custom ActiveRecord Type (cleanest approach)

```ruby
class CoordinateType < ActiveRecord::Type::Value
  def cast(value)
    case value
    when Coordinate then value
    when Hash then Coordinate.new(**value.symbolize_keys)
    when String then Coordinate.new(**JSON.parse(value).symbolize_keys)
    end
  end

  def serialize(value) = value&.to_h&.to_json

  def deserialize(value)
    return nil unless value
    Coordinate.new(**JSON.parse(value).symbolize_keys)
  end
end

ActiveRecord::Type.register(:coordinate, CoordinateType)

class Event < ApplicationRecord
  attribute :location, :coordinate
end
```

### JSON Column (simpler, less type-safe)

```ruby
class Event < ApplicationRecord
  # Migration: add_column :events, :location, :jsonb
  def location
    @location ||= begin
      data = read_attribute(:location)
      data ? Coordinate.new(**data.symbolize_keys) : nil
    end
  end

  def location=(coord)
    @location = coord
    write_attribute(:location, coord&.to_h)
  end
end
```

## Key Rules

- Always implement `==`, `eql?`, and `hash` for non-Struct/Data value objects
- Always implement `to_s` for debugging
- Validate invariants in `initialize` (e.g., end_date > start_date)
- Make value objects immutable — create new instances instead of mutating
- Don't add behavior unrelated to the value's domain concept
