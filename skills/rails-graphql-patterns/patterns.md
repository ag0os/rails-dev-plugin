# GraphQL Patterns for Rails — Additive Details

Non-obvious patterns and reusable sources for graphql-ruby. For standard type definitions, input types, enums, query types, and basic CRUD mutations, follow graphql-ruby conventions.

## DataLoader Source Variants

### AssociationLoader (has_many / has_one)

Batch-preloads ActiveRecord associations to avoid N+1 on collection fields.

```ruby
# app/graphql/sources/association_loader.rb
class Sources::AssociationLoader < GraphQL::Dataloader::Source
  def initialize(model_class, association_name)
    @model_class = model_class
    @association_name = association_name
  end

  def fetch(records)
    ActiveRecord::Associations::Preloader.new(
      records: records,
      associations: @association_name
    ).call

    records.map { |record| record.public_send(@association_name) }
  end
end

# Usage
def comments
  dataloader.with(Sources::AssociationLoader, Post, :comments).load(object)
end
```

### CountLoader (aggregate counts without loading records)

```ruby
# app/graphql/sources/count_loader.rb
class Sources::CountLoader < GraphQL::Dataloader::Source
  def initialize(model_class, foreign_key)
    @model_class = model_class
    @foreign_key = foreign_key
  end

  def fetch(ids)
    counts = @model_class
      .where(@foreign_key => ids)
      .group(@foreign_key)
      .count

    ids.map { |id| counts[id] || 0 }
  end
end

# Usage
def comments_count
  dataloader.with(Sources::CountLoader, Comment, :post_id).load(object.id)
end
```

## BaseMutation with Pundit Authorization

Opinionated base mutation wiring Pundit policy checks and structured user errors.

```ruby
module Mutations
  class BaseMutation < GraphQL::Schema::RelayClassicMutation
    argument_class Types::BaseArgument
    field_class Types::BaseField
    input_object_class Types::BaseInputObject
    object_class Types::BaseObject

    def current_user
      context[:current_user]
    end

    def authenticate!
      raise GraphQL::ExecutionError, "Authentication required" unless current_user
    end

    def authorize!(record, action)
      policy = Pundit.policy!(current_user, record)
      unless policy.public_send("#{action}?")
        raise GraphQL::ExecutionError, "Not authorized"
      end
    end
  end
end
```

## Structured User Errors

Return field-level errors from mutations instead of raising exceptions.

```ruby
module Types
  class UserErrorType < Types::BaseObject
    field :field, String, description: "Field with error (camelCase)"
    field :message, String, null: false, description: "Error message"
  end
end

# In mutation resolve methods
def user_errors(record)
  record.errors.map do |error|
    { field: error.attribute.to_s.camelize(:lower), message: error.message }
  end
end
```

## Subscriptions — Filtering by Argument

```ruby
module Types
  class SubscriptionType < Types::BaseObject
    field :post_created, Types::PostType, null: false do
      argument :author_id, ID, required: false
    end

    def post_created(author_id: nil)
      return object unless author_id
      object if object.author_id.to_s == author_id
    end
  end
end
```

## Testing Pattern

Execute queries against the schema directly. Assert on `result["data"]` and `result["errors"]`.

```ruby
RSpec.describe Types::QueryType do
  let(:user) { create(:user) }
  let(:context) { { current_user: user } }

  describe 'posts query' do
    let(:query) do
      <<~GQL
        query($status: PostStatus) {
          posts(status: $status) {
            nodes { id title }
            totalCount
          }
        }
      GQL
    end

    it 'returns published posts' do
      create_list(:post, 3, :published)
      result = MyAppSchema.execute(query, context: context, variables: { status: "PUBLISHED" })

      expect(result['data']['posts']['totalCount']).to eq(3)
      expect(result['errors']).to be_nil
    end
  end
end
```
