---
name: rails-testing-patterns
description: Analyzes Rails test suites and recommends testing best practices for RSpec and Minitest. Use when writing new tests, reviewing test coverage, fixing flaky tests, improving test performance, choosing between test types (unit, integration, system, request), or setting up factories and fixtures. NOT for production monitoring, deployment verification, or load/stress testing infrastructure.
allowed-tools: Read, Grep, Glob
---

# Rails Testing Patterns

Analyze and recommend best practices for Rails test suites. Covers both RSpec and Minitest, all test types, factories, fixtures, and performance.

See [patterns.md](patterns.md) for detailed code examples for both frameworks.

## Quick Reference

| Test Type | Purpose | Speed | Framework Files |
|-----------|---------|-------|-----------------|
| Unit (Model) | Validations, scopes, methods | Fast | `test/models/` or `spec/models/` |
| Request/Controller | HTTP responses, params, auth | Medium | `test/controllers/` or `spec/requests/` |
| Integration | Multi-step workflows | Medium | `test/integration/` or `spec/features/` |
| System | Full browser simulation | Slow | `test/system/` or `spec/system/` |
| Mailer | Email content and delivery | Fast | `test/mailers/` or `spec/mailers/` |
| Job | Background job behavior | Fast | `test/jobs/` or `spec/jobs/` |

## Core Principles

1. **Arrange-Act-Assert** -- every test has three clear phases
2. **Test behavior, not implementation** -- assert outcomes, not internal calls
3. **One assertion per concept** -- each test validates a single behavior
4. **Minimal test data** -- create only what the test needs
5. **No test interdependence** -- tests must pass in any order
6. **Fast feedback** -- unit tests < 100ms, system tests < 5s

## Key Patterns

### RSpec Model Spec

```ruby
RSpec.describe User, type: :model do
  describe "validations" do
    it { should validate_presence_of(:email) }
    it { should validate_uniqueness_of(:email).case_insensitive }
  end

  describe "#full_name" do
    let(:user) { build(:user, first_name: "Jane", last_name: "Doe") }

    it "returns combined first and last name" do
      expect(user.full_name).to eq("Jane Doe")
    end
  end
end
```

### Minitest Model Test

```ruby
class UserTest < ActiveSupport::TestCase
  test "should not save without email" do
    user = User.new
    assert_not user.save, "Saved user without email"
  end

  test "full_name returns combined name" do
    user = User.new(first_name: "Jane", last_name: "Doe")
    assert_equal "Jane Doe", user.full_name
  end
end
```

### Request Spec (RSpec)

```ruby
RSpec.describe "Products API", type: :request do
  describe "GET /api/v1/products" do
    let!(:products) { create_list(:product, 3) }

    it "returns all products with 200" do
      get "/api/v1/products", headers: auth_headers
      expect(response).to have_http_status(:ok)
      expect(json_response.size).to eq(3)
    end
  end
end
```

### Integration Test (Minitest)

```ruby
class UsersControllerTest < ActionDispatch::IntegrationTest
  test "should create user" do
    assert_difference("User.count") do
      post users_url, params: { user: { email: "new@example.com" } }
    end
    assert_redirected_to user_url(User.last)
  end
end
```

### System Test

```ruby
# Works in both RSpec (type: :system) and Minitest (< ApplicationSystemTestCase)
test "user can sign up" do
  visit new_user_registration_path
  fill_in "Email", with: "test@example.com"
  fill_in "Password", with: "password123"
  fill_in "Password confirmation", with: "password123"
  click_button "Sign up"
  assert_text "Welcome!"
end
```

## Anti-Patterns

| Bad | Good | Why |
|-----|------|-----|
| Testing Rails internals | Test your code only | Rails is already tested |
| `sleep 2` in system tests | Use `have_content` / `assert_text` waits | Capybara auto-waits |
| Shared mutable state | `let` / `setup` per test | Prevents coupling |
| Testing private methods | Test through public interface | Implementation may change |
| Huge factory with all fields | Minimal factory + traits | Clarity and speed |
| No edge-case tests | Test nil, empty, boundary | Catches real bugs |

## Test Data Strategy

| Approach | When | Tool |
|----------|------|------|
| Factories | Dynamic, unique data per test | FactoryBot |
| Fixtures | Static reference data, fast | YAML fixtures |
| Traits | Variations of a factory | `factory :user, trait: :admin` |
| Sequences | Unique values (emails, slugs) | `sequence(:email)` |

## Edge Cases to Always Test

- Nil / blank / empty inputs
- Boundary values (0, -1, MAX)
- Unauthorized access attempts
- Invalid parameter combinations
- Concurrent modifications (where relevant)

## Output Format

When reporting on test quality, use:

```
## Test Analysis: [spec_or_test_file]

**Coverage Gaps:**
- [model/action] missing test for [scenario]

**Issues:**
- [severity] description

**Recommendations:**
1. actionable recommendation
```
