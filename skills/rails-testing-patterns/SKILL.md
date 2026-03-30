---
name: rails-testing-patterns
description: Analyzes Rails test suites and recommends testing best practices for RSpec and Minitest. Use when writing new tests, reviewing test coverage, fixing flaky tests, improving test performance, choosing between test types (unit, integration, system, request), or setting up factories and fixtures. NOT for production monitoring, deployment verification, or load/stress testing infrastructure.
allowed-tools: Read, Grep, Glob
---

# Rails Testing Patterns

Use standard RSpec and Minitest conventions for syntax, assertions, matchers, and Capybara DSL. This skill focuses on **opinionated choices** — what to test, which framework to use, and how to structure test data.

See [patterns.md](patterns.md) for decision guidance and non-obvious patterns.

## Test Type Decision Matrix

| Test Type | When to Write | Speed | ROI Notes |
|-----------|--------------|-------|-----------|
| Unit (Model) | Business logic, validations, scopes, custom methods | Fast | Highest ROI — write these first in service-oriented stacks |
| Request | API endpoints, auth flows, param handling | Medium | Highest ROI for api-first stacks; prefer over controller specs in RSpec |
| Integration | Multi-step workflows spanning controllers | Medium | Omakase priority — tests the full stack as Rails intends |
| System | Critical user journeys only (sign-up, checkout, onboarding) | Slow | Limit to 10-20 per app; never test CRUD forms with system tests |
| Mailer | Non-trivial email content, conditional delivery | Fast | Skip if mailer is just a default scaffold |
| Job | Idempotency, retry behavior, side effects | Fast | Always test jobs that touch external services |

## Profile-Aware Testing

**Detect the project's profile before recommending a testing approach.**

| Decision | Omakase | Service-Oriented | API-First |
|----------|---------|-----------------|-----------|
| Framework | Minitest | RSpec | RSpec |
| Test data | Fixtures | FactoryBot | FactoryBot |
| Directory | `test/` | `spec/` | `spec/` |
| First tests to write | Integration/controller tests | Unit tests + request specs | Request specs |
| System tests | Yes, for key flows | Sparingly | No — use request specs |

## Anti-Patterns

| Anti-Pattern | Do Instead | Why |
|-------------|-----------|-----|
| Testing Rails internals (validates_presence_of works?) | Test your domain logic only | Rails is already tested |
| `sleep` in system tests | Rely on Capybara's built-in waiting (`have_content`, `assert_text`) | Sleep is flaky and slow |
| Shared mutable state between tests | Fresh state per test via `let`/`setup` | Test ordering bugs are the hardest to debug |
| Testing private methods directly | Test through the public interface | Couples tests to implementation |
| Huge factory/fixture with every field | Minimal defaults + traits/overrides | Reduces test fragility and speeds diagnosis |
| System tests for CRUD forms | Request/integration tests | 10x faster, same coverage for form submissions |
| Mocking the object under test | Mock collaborators, not the subject | Tests nothing useful |

## Test Data Strategy

**Omakase — fixtures first:**
- Fixtures are loaded once per test run (fast) and double as development seed data
- Use **realistic, named** fixtures: `jane:`, `admin:` — never `user_1:`, `user_2:`
- Use inline `Model.new(...)` only for one-off edge cases not worth a fixture
- Fixture files are the canonical source of test data — keep them curated

**Service-oriented / API-first — factories first:**
- Minimal factory defaults — only required fields, everything else via traits
- Use `build` (not `create`) whenever persistence isn't needed — dramatically faster
- Use `sequence` for uniqueness constraints (emails, slugs)
- Avoid deeply nested `create` chains — if a test needs 4+ `create` calls, consider fixtures for that scenario
- Reserve fixtures for truly static reference data (countries, currencies)

## Edge Cases to Always Test

- Nil / blank / empty inputs
- Boundary values (0, -1, MAX)
- Unauthorized and forbidden access
- Invalid parameter combinations
- Concurrent modifications (optimistic locking, uniqueness races)

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
