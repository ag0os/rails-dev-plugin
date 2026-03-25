# Rails Testing Patterns -- Detailed Reference

## RSpec Patterns

### Factory Setup (FactoryBot)

```ruby
# spec/factories/users.rb
FactoryBot.define do
  factory :user do
    sequence(:email) { |n| "user#{n}@example.com" }
    first_name { "Jane" }
    last_name  { "Doe" }
    password   { "password123" }

    trait :admin do
      role { :admin }
    end

    trait :with_posts do
      transient do
        posts_count { 3 }
      end

      after(:create) do |user, evaluator|
        create_list(:post, evaluator.posts_count, author: user)
      end
    end
  end
end
```

### Model Specs

```ruby
RSpec.describe Product, type: :model do
  describe "validations" do
    it { should validate_presence_of(:name) }
    it { should validate_numericality_of(:price).is_greater_than(0) }
  end

  describe "associations" do
    it { should belong_to(:category) }
    it { should have_many(:reviews).dependent(:destroy) }
  end

  describe "scopes" do
    describe ".active" do
      let!(:active)   { create(:product, active: true) }
      let!(:inactive) { create(:product, active: false) }

      it "returns only active products" do
        expect(Product.active).to contain_exactly(active)
      end
    end
  end

  describe "#discounted_price" do
    subject(:product) { build(:product, price: 100) }

    it "applies the discount percentage" do
      expect(product.discounted_price(10)).to eq(90)
    end

    it "returns full price for zero discount" do
      expect(product.discounted_price(0)).to eq(100)
    end

    it "raises for negative discount" do
      expect { product.discounted_price(-5) }.to raise_error(ArgumentError)
    end
  end
end
```

### Request Specs

```ruby
RSpec.describe "Api::V1::Products", type: :request do
  let(:user)         { create(:user) }
  let(:auth_headers) { { "Authorization" => "Bearer #{jwt_for(user)}" } }

  describe "GET /api/v1/products" do
    let!(:products) { create_list(:product, 3) }

    it "returns paginated products" do
      get "/api/v1/products", params: { page: 1, per_page: 2 }, headers: auth_headers
      expect(response).to have_http_status(:ok)
      expect(json_response["data"].size).to eq(2)
      expect(json_response["meta"]["total"]).to eq(3)
    end
  end

  describe "POST /api/v1/products" do
    let(:valid_params) { { product: { name: "Widget", price: 9.99 } } }

    it "creates a product" do
      expect {
        post "/api/v1/products", params: valid_params, headers: auth_headers
      }.to change(Product, :count).by(1)

      expect(response).to have_http_status(:created)
    end

    context "with invalid params" do
      it "returns validation errors" do
        post "/api/v1/products", params: { product: { name: "" } }, headers: auth_headers
        expect(response).to have_http_status(:unprocessable_entity)
        expect(json_response["errors"]).to have_key("name")
      end
    end
  end
end
```

### System Specs

```ruby
RSpec.describe "User Registration", type: :system do
  before { driven_by(:selenium_chrome_headless) }

  it "allows a new user to sign up" do
    visit new_user_registration_path

    fill_in "Email", with: "newuser@example.com"
    fill_in "Password", with: "securepassword"
    fill_in "Password confirmation", with: "securepassword"
    click_button "Sign up"

    expect(page).to have_content("Welcome!")
    expect(User.last.email).to eq("newuser@example.com")
  end

  it "shows errors for invalid input" do
    visit new_user_registration_path
    click_button "Sign up"

    expect(page).to have_content("can't be blank")
  end
end
```

### Shared Examples

```ruby
RSpec.shared_examples "a paginatable resource" do |resource_path|
  it "respects per_page parameter" do
    get resource_path, params: { per_page: 1 }, headers: auth_headers
    expect(json_response["data"].size).to eq(1)
  end

  it "includes pagination meta" do
    get resource_path, headers: auth_headers
    expect(json_response["meta"]).to include("total", "page", "per_page")
  end
end
```

### Mocking External Services

```ruby
RSpec.describe PaymentService do
  describe "#charge" do
    let(:gateway) { instance_double(Stripe::PaymentIntent) }

    before do
      allow(Stripe::PaymentIntent).to receive(:create).and_return(gateway)
      allow(gateway).to receive(:status).and_return("succeeded")
    end

    it "returns success for valid charge" do
      result = described_class.new.charge(amount: 1000, token: "tok_visa")
      expect(result).to be_success
    end
  end
end
```

## Minitest Patterns

### Fixtures

```yaml
# test/fixtures/users.yml
admin:
  email: admin@example.com
  first_name: Admin
  last_name: User
  role: admin

regular:
  email: regular@example.com
  first_name: Regular
  last_name: User
  role: member
```

### Model Tests

```ruby
class ProductTest < ActiveSupport::TestCase
  test "valid product saves" do
    product = Product.new(name: "Widget", price: 9.99, category: categories(:electronics))
    assert product.valid?
  end

  test "requires name" do
    product = Product.new(price: 9.99)
    assert_not product.valid?
    assert_includes product.errors[:name], "can't be blank"
  end

  test "price must be positive" do
    product = Product.new(name: "Widget", price: -1)
    assert_not product.valid?
  end

  test "active scope returns only active products" do
    active_count = Product.where(active: true).count
    assert_equal active_count, Product.active.count
  end
end
```

### Controller / Integration Tests

```ruby
class ProductsControllerTest < ActionDispatch::IntegrationTest
  setup do
    @product = products(:widget)
    sign_in users(:admin)
  end

  test "should get index" do
    get products_url
    assert_response :success
    assert_select "h1", "Products"
  end

  test "should create product" do
    assert_difference("Product.count") do
      post products_url, params: { product: { name: "New", price: 5.00 } }
    end
    assert_redirected_to product_url(Product.last)
    follow_redirect!
    assert_select ".flash", /created/i
  end

  test "should not create invalid product" do
    assert_no_difference("Product.count") do
      post products_url, params: { product: { name: "" } }
    end
    assert_response :unprocessable_entity
  end
end
```

### System Tests (Minitest)

```ruby
class UserSignupTest < ApplicationSystemTestCase
  driven_by :selenium, using: :headless_chrome

  test "visiting the sign up page and registering" do
    visit new_user_registration_url
    fill_in "Email", with: "newuser@example.com"
    fill_in "Password", with: "password123"
    fill_in "Password confirmation", with: "password123"
    click_on "Sign up"

    assert_text "Welcome!"
  end
end
```

### Custom Assertions

```ruby
# test/support/custom_assertions.rb
module CustomAssertions
  def assert_json_response(status: :ok)
    assert_response status
    assert_equal "application/json", response.media_type
    JSON.parse(response.body)
  end
end

# In test_helper.rb
class ActiveSupport::TestCase
  include CustomAssertions
end
```

## Performance Patterns

### Parallel Tests

```ruby
# RSpec: spec/spec_helper.rb
RSpec.configure do |config|
  config.parallelize(workers: :number_of_processors) # Rails 7+
end

# Minitest: test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
end
```

### Database Cleaner (RSpec)

```ruby
RSpec.configure do |config|
  config.use_transactional_fixtures = true

  config.before(:each, type: :system) do
    driven_by :selenium_chrome_headless
  end
end
```

### Stubbing Time

```ruby
# RSpec
it "expires after 24 hours" do
  travel_to Time.zone.parse("2025-01-01 12:00") do
    token = create(:token)
    travel 25.hours
    expect(token).to be_expired
  end
end

# Minitest
test "expires after 24 hours" do
  travel_to Time.zone.parse("2025-01-01 12:00") do
    token = tokens(:expiring)
    travel 25.hours
    assert token.expired?
  end
end
```

### VCR for External APIs

```ruby
RSpec.describe "GitHub API", :vcr do
  it "fetches repository info" do
    result = GitHubClient.new.repo("rails/rails")
    expect(result[:name]).to eq("rails")
  end
end
```

## Edge Case Testing Patterns

### Nil / Empty

```ruby
it "handles nil gracefully" do
  expect(described_class.new(name: nil).display_name).to eq("Unknown")
end

it "handles empty string" do
  expect(described_class.new(name: "").display_name).to eq("Unknown")
end
```

### Boundary Values

```ruby
describe "age validation" do
  it { expect(build(:user, age: 0)).not_to be_valid }
  it { expect(build(:user, age: 1)).to be_valid }
  it { expect(build(:user, age: 150)).to be_valid }
  it { expect(build(:user, age: 151)).not_to be_valid }
end
```

### Authorization

```ruby
context "when not authenticated" do
  it "returns 401" do
    get "/api/v1/products"
    expect(response).to have_http_status(:unauthorized)
  end
end

context "when not authorized" do
  let(:user) { create(:user, role: :viewer) }

  it "returns 403" do
    delete "/api/v1/products/#{product.id}", headers: auth_headers
    expect(response).to have_http_status(:forbidden)
  end
end
```

## Test Helpers

### JSON Response Helper

```ruby
# spec/support/json_helpers.rb
module JsonHelpers
  def json_response
    @json_response ||= JSON.parse(response.body, symbolize_names: true)
  end
end

RSpec.configure do |config|
  config.include JsonHelpers, type: :request
end
```

### Authentication Helper

```ruby
# spec/support/auth_helpers.rb
module AuthHelpers
  def jwt_for(user)
    JWT.encode({ user_id: user.id, exp: 1.hour.from_now.to_i },
               Rails.application.secret_key_base)
  end

  def auth_headers_for(user)
    { "Authorization" => "Bearer #{jwt_for(user)}" }
  end
end
```
