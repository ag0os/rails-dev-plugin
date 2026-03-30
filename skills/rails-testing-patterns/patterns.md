# Rails Testing Patterns -- Decision Guide & Non-Obvious Patterns

Use standard RSpec/Minitest syntax and conventions for basic test structure. This document covers only the **decisions and patterns that aren't obvious**.

## Fixture vs Factory: The Real Trade-offs

### When Fixtures Win (Not Just "Omakase Default")

- **Speed at scale**: 500+ factories in a suite add minutes; fixtures load once via bulk insert
- **Referential integrity**: Fixtures enforce FK relationships at load time — broken references fail fast
- **Readability for stable domains**: When your User/Account/Role models rarely change, named fixtures are clearer than factory chains

### When Factories Win (Not Just "RSpec Default")

- **Rapidly changing schemas**: Factories adapt with one default change; fixtures need N files updated
- **Combinatorial test data**: Traits compose (`create(:user, :admin, :with_posts, :deactivated)`) — fixtures require a new named entry per combination
- **Test isolation**: Each test gets exactly the data it needs, nothing inherited from a shared YAML file

### Hybrid Approach (Often Best)

```ruby
# Fixtures for stable reference data
# test/fixtures/roles.yml or spec/fixtures/roles.yml
admin:
  name: admin
  permissions: manage_all

member:
  name: member
  permissions: read_only

# Factories for test-specific transactional data
FactoryBot.define do
  factory :order do
    user
    status { :pending }
    trait :completed do
      status { :completed }
      completed_at { 1.hour.ago }
    end
  end
end
```

## Performance Patterns That Actually Matter

### `build` vs `create` — The Biggest Quick Win

```ruby
# SLOW: hits the database for a validation test
it "requires an email" do
  user = create(:user, email: nil)  # INSERT then validate
  expect(user).not_to be_valid
end

# FAST: no database round-trip
it "requires an email" do
  user = build(:user, email: nil)
  expect(user).not_to be_valid
end
```

Rule: use `create` only when the test needs a persisted record (associations, queries, controller tests).

### Parallel Tests — Pitfalls

```ruby
# Both frameworks support parallelization, but watch for:
# 1. Shared database state — fixtures are safe; factories need transactional isolation
# 2. Non-transactional side effects — file writes, cache keys, ENV mutations
# 3. Port conflicts — system tests need unique ports per worker

# Minitest (built-in)
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
  parallelize_setup do |worker|
    ActiveStorage::Blob.service.root = "#{ActiveStorage::Blob.service.root}-#{worker}"
  end
end
```

### Transactional Tests vs Database Cleaner

- Rails' default `use_transactional_tests = true` (Minitest) / `use_transactional_fixtures = true` (RSpec) is sufficient for 95% of cases
- Only add DatabaseCleaner when you have **non-transactional tests** (system tests with separate browser thread, tests that explicitly commit)
- Truncation strategy is 5-10x slower than transactions — never use it globally

## Mocking: When and What

### Mock Boundaries, Not Internals

```ruby
# BAD: mocking ActiveRecord — tests nothing real
allow(User).to receive(:find).and_return(user)

# GOOD: mocking an external service boundary
allow(Stripe::PaymentIntent).to receive(:create).and_return(
  double(id: "pi_123", status: "succeeded")
)
```

### The WebMock/VCR Decision

| Approach | Use When | Watch Out For |
|----------|----------|---------------|
| WebMock stubs | You control the exact request/response | Stubs drift from real API over time |
| VCR cassettes | Recording real API responses | Cassettes contain secrets — add to `.gitignore` or sanitize |
| Fake server (e.g., `fake_stripe`) | Heavy integration with one service | Maintenance cost of the fake |

**Prefer WebMock for unit/request tests; VCR for integration tests that touch real APIs during recording.**

## Test Helpers Worth Writing

### JSON Response Helper (Every API Project Needs This)

```ruby
# One canonical helper — avoid re-parsing response.body in every test
module JsonHelpers
  def json_response
    @json_response ||= JSON.parse(response.body, symbolize_names: true)
  end
end

# RSpec: config.include JsonHelpers, type: :request
# Minitest: include JsonHelpers in ActionDispatch::IntegrationTest
```

### Authentication Test Helper

```ruby
# Adapt to your auth strategy — this is JWT; for session-based, use sign_in helper
module AuthHelpers
  def auth_headers_for(user)
    token = JWT.encode(
      { user_id: user.id, exp: 1.hour.from_now.to_i },
      Rails.application.secret_key_base
    )
    { "Authorization" => "Bearer #{token}" }
  end
end
```

## System Tests: Keeping Them Fast and Stable

- **Limit scope**: Only test JavaScript-dependent flows and multi-page journeys
- **Use headless Chrome**: `driven_by :selenium, using: :headless_chrome`
- **Never `sleep`**: Capybara auto-waits on `have_content`, `have_selector`, `assert_text`
- **Avoid exact text matching**: Prefer `have_content("Welcome")` over `have_content("Welcome, Jane Doe! Your account was created on March 30, 2026.")`
- **One browser session per test**: Never carry state between system test examples
- **Screenshot on failure** (built into Rails 7+): Helps debug CI failures without re-running

## Flaky Test Diagnosis

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Passes alone, fails in suite | Shared state (class variable, global, cache) | Isolate state in `setup`/`before`; check `Rails.cache.clear` |
| Fails intermittently on CI | Timing/ordering dependency | Run with `--seed random`; add `parallelize_teardown` cleanup |
| System test timeout | Slow JS or missing wait | Replace `sleep` with Capybara finders; increase `Capybara.default_max_wait_time` |
| Different results per worker | Database state leaking | Ensure transactional fixtures; check for raw SQL that bypasses AR transactions |
