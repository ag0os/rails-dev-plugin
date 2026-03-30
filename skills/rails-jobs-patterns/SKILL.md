---
name: rails-jobs-patterns
description: Analyzes and recommends ActiveJob background processing patterns including idempotency, retry strategies, batch processing, scheduled jobs, and queue management. Use when creating background jobs, configuring Sidekiq, handling async workflows, or optimizing job performance. NOT for synchronous controller actions, model callbacks, service object design, or real-time Turbo Streams.
allowed-tools: Read, Grep, Glob
---

# Rails Background Job Patterns

Analyze and recommend patterns for reliable, efficient background jobs in Rails applications.

Follow standard Rails conventions for `retry_on`, `discard_on`, `queue_as`, and basic ActiveJob structure. Focus on the opinionated patterns below.

## Quick Reference

| Pattern | Use When |
|---------|----------|
| `_later`/`_now` convention | Every async operation — name both versions |
| Shallow job + model logic | All new jobs — jobs are wrappers only |
| Double-check locking | Idempotency with concurrent workers |
| Self-splitting batch | Datasets too large for one job |
| `limits_concurrency` | Solid Queue race condition prevention |
| `perform_all_later` | Bulk enqueueing (Rails 7.1+) |

## Supporting Documentation

- [patterns.md](patterns.md) - Complete job patterns with detailed examples

## Core Principles

1. **Jobs are shallow wrappers**: Business logic lives in models (omakase) or service objects (service-oriented). Jobs handle only queuing and retry infrastructure
2. **`_later`/`_now` convention**: Define `thing_later` (queues job) and `thing` (does work) on the model. The job just calls the model method
3. **Double-check locking for idempotency**: Guard clause THEN `with_lock` THEN guard again — prevents races between concurrent workers
4. **Pass IDs, not objects**: Avoid serialization issues and stale data
5. **Queue segmentation**: Separate critical, default, low, and mailers

## The `_later`/`_now` Pattern (37signals)

```ruby
# Model defines both versions
class Webhook::Delivery < ApplicationRecord
  after_create_commit :deliver_later

  def deliver_later
    Webhook::DeliveryJob.perform_later(self)
  end

  def deliver
    in_progress!
    self.response = perform_request
    completed!
  rescue => e
    failed!(e.message)
  end
end

# Job is a shallow wrapper — no business logic
class Webhook::DeliveryJob < ApplicationJob
  queue_as :webhooks
  def perform(delivery)
    delivery.deliver
  end
end
```

## Idempotency: Double-Check Locking

The critical pattern — guard clause alone has a race window. Always combine with `with_lock`:

```ruby
def perform(import_id)
  import = Import.find(import_id)
  return if import.completed?          # Fast path: skip lock if done

  import.with_lock do
    return if import.completed?        # Safe path: re-check under lock
    process_import(import)
    import.update!(status: "completed")
  end
end
```

## Self-Splitting Batch Jobs

For datasets too large for one job — the job re-enqueues itself for the next chunk:

```ruby
class LargeImportJob < ApplicationJob
  BATCH_SIZE = 1_000

  def perform(dataset_id, offset = 0)
    records = Dataset.find(dataset_id).records.offset(offset).limit(BATCH_SIZE)
    return if records.empty?

    process_batch(records)
    self.class.perform_later(dataset_id, offset + BATCH_SIZE)
  end
end
```

## Concurrency Control

### Solid Queue — `limits_concurrency`

```ruby
class Storage::MaterializeJob < ApplicationJob
  queue_as :backend
  limits_concurrency to: 1, key: ->(owner) { owner }

  def perform(owner)
    owner.materialize_storage
  end
end
```

### Sidekiq — `sidekiq-unique-jobs`

```ruby
class SyncUserJob < ApplicationJob
  sidekiq_options lock: :until_executed,
                  lock_args_method: ->(args) { [args.first] }

  def perform(user_id)
    UserSyncService.new(user_id).sync!
  end
end
```

Lock types: `:until_executing` (unique in queue), `:until_executed` (through completion), `:until_and_while_executing` (most restrictive).

## Bulk Enqueueing (Rails 7.1+)

```ruby
class Notification::Bundle
  def self.deliver_all_later
    due.find_in_batches do |batch|
      jobs = batch.map { |bundle| DeliverJob.new(bundle) }
      ActiveJob.perform_all_later(jobs)  # Single DB operation
    end
  end
end
```

## Multi-Tenant Context Serialization

For apps using `CurrentAttributes` with multi-tenancy — automatically serialize account context across job execution. See [patterns.md](patterns.md) for the `FizzyActiveJobExtensions` pattern that captures `Current.account` on enqueue and restores it on perform.

**Skip this pattern for single-tenant apps.**

## Anti-Patterns

| Anti-Pattern | Fix |
|-------------|-----|
| Business logic inside job | **Omakase:** delegate to model. **Service-oriented:** delegate to service |
| Guard clause without `with_lock` | Use double-check locking pattern above |
| `find_in_batches` all in one job | Self-splitting batch or `perform_all_later` |
| No queue segmentation | Use priority queues (critical/default/low/mailers) |

## Context Detection

| Check | Command | Implication |
|-------|---------|-------------|
| Job adapter | `grep "queue_adapter" config/` | Solid Queue vs Sidekiq patterns |
| Multi-tenancy | `grep "Current.account" app/` | Context serialization needed |
| Rails version | `grep "rails " Gemfile.lock` | `perform_all_later` available 7.1+ |
| Recurring jobs | Check `config/recurring.yml` or `sidekiq_schedule.yml` | Respect existing scheduler |

## Output Format

When analyzing or creating jobs, provide:
1. **Job file** in `app/jobs/` with retry/discard configuration
2. **Idempotency strategy** (double-check locking, unique constraint)
3. **Queue assignment** with rationale
4. **Test outline** using `ActiveJob::TestHelper`
5. **Monitoring** notes (logging, metrics, alerting)
