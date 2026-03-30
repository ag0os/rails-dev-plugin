# Background Job Patterns — Detailed Reference

Follow standard Rails conventions for basic ActiveJob structure, `retry_on`/`discard_on` syntax, and queue configuration. This file covers opinionated and non-obvious patterns.

## The `_later`/`_now` Convention (37signals Pattern)

Define both `_later` (queues job) and the synchronous method on the model. Jobs become shallow wrappers.

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

# Job is a shallow wrapper
class Webhook::DeliveryJob < ApplicationJob
  queue_as :webhooks
  def perform(delivery)
    delivery.deliver
  end
end
```

**Benefits**: Business logic testable without job infrastructure, clear naming for async operations, easy to call synchronously in tests or console.

## Shallow Jobs Delegating to Models (37signals Pattern)

```ruby
# PREFERRED: Shallow job
class Notification::Bundle::DeliverJob < ApplicationJob
  include SmtpDeliveryErrorHandling
  queue_as :backend

  def perform(bundle)
    bundle.deliver  # All logic in the model
  end
end

# Model contains the business logic
class Notification::Bundle < ApplicationRecord
  def deliver
    return if delivered?
    recipients.each do |recipient|
      NotificationMailer.bundle(self, recipient).deliver_now
    end
    update!(delivered_at: Time.current)
  end
end

# AVOID: Fat job with business logic
class NotificationDeliverJob < ApplicationJob
  def perform(notification)
    user = notification.user
    if user.email_verified?
      NotificationMailer.deliver(notification).deliver_now
      notification.update!(delivered_at: Time.current)
    end
  end
end
```

## Idempotency: Double-Check Locking

The guard-then-lock-then-guard-again pattern prevents races between concurrent workers:

```ruby
class ImportDataJob < ApplicationJob
  def perform(import_id)
    import = Import.find(import_id)
    return if import.completed?        # Fast path

    import.with_lock do
      return if import.completed?      # Safe path under lock
      process_import(import)
      import.update!(status: 'completed', completed_at: Time.current)
    end
  end
end
```

**Why both checks?** The first avoids an unnecessary lock acquisition. The second prevents the race where two workers both pass the first check before either acquires the lock.

## Concurrency Control

### Solid Queue — `limits_concurrency`

```ruby
class Storage::MaterializeJob < ApplicationJob
  queue_as :backend
  limits_concurrency to: 1, key: ->(owner) { owner }
  discard_on ActiveJob::DeserializationError

  def perform(owner)
    owner.materialize_storage
  end
end
# Multiple jobs for the same owner queue; different owners run in parallel
```

**Use cases**: Prevent race conditions on shared resources, limit API rate per account, control DB load per tenant.

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

Lock types:
- `:until_executing` — Unique while in queue, allows retries
- `:until_executed` — Unique through completion
- `:until_and_while_executing` — Most restrictive

## Self-Splitting Batch Jobs

For datasets too large to process in a single job execution:

```ruby
class LargeDataProcessJob < ApplicationJob
  BATCH_SIZE = 1000

  def perform(dataset_id, offset = 0)
    dataset = Dataset.find(dataset_id)
    batch = dataset.records.offset(offset).limit(BATCH_SIZE)
    return if batch.empty?

    process_batch(batch)
    self.class.perform_later(dataset_id, offset + BATCH_SIZE)
  end
end
```

## Efficient Batch Enqueueing (Rails 7.1+)

Use `ActiveJob.perform_all_later` to enqueue multiple jobs in a single database operation:

```ruby
class Notification::Bundle
  class << self
    def deliver_all_later
      due.find_in_batches do |batch|
        jobs = batch.map { |bundle| DeliverJob.new(bundle) }
        ActiveJob.perform_all_later(jobs)
      end
    end
  end
end
```

## Scheduled Jobs

### Solid Queue Recurring Jobs

```yaml
# config/recurring.yml
production: &production
  deliver_bundled_notifications:
    command: "Notification::Bundle.deliver_all_later"
    schedule: every 30 minutes

  auto_postpone_all_due:
    command: "Card.auto_postpone_all_due"
    schedule: every hour at minute 50

  delete_unused_tags:
    class: DeleteUnusedTagsJob
    schedule: every day at 04:02

development:
  <<: *production
```

### Recurring Job Idempotency

Always guard recurring jobs against duplicate execution:

```ruby
class DailyReportJob < ApplicationJob
  def perform(date = Date.current)
    return if Report.exists?(date: date, type: 'daily')
    report = Report.create!(date: date, type: 'daily', data: generate_report_data(date))
    ReportMailer.daily_report(report).deliver_later
  end
end
```

## Multi-Tenant Context Serialization (37signals Pattern)

For multi-tenant apps using `CurrentAttributes` — automatically serialize and restore account context:

```ruby
# config/initializers/active_job_extensions.rb
module FizzyActiveJobExtensions
  extend ActiveSupport::Concern

  prepended do
    attr_reader :account
    self.enqueue_after_transaction_commit = true
  end

  def initialize(...)
    super
    @account = Current.account
  end

  def serialize
    super.merge({ "account" => @account&.to_gid })
  end

  def deserialize(job_data)
    super
    if _account = job_data.fetch("account", nil)
      @account = GlobalID::Locator.locate(_account)
    end
  end

  def perform_now
    if account.present?
      Current.with_account(account) { super }
    else
      super
    end
  end
end

ActiveSupport.on_load(:active_job) do
  prepend FizzyActiveJobExtensions
end
```

**Benefits**: No manual account passing, prevents cross-tenant data leaks, works with existing `Current` pattern.

**Skip for single-tenant apps.** Detection: check for `app/models/current.rb` and `account_id` in schema.

## Context Detection Guide

| Pattern | Detection | Applicability |
|---------|-----------|---------------|
| `_later`/`_now` convention | Always | All projects |
| Shallow jobs | Always for new jobs | All projects |
| Context serialization | `Current.account` + multi-tenant schema | Multi-tenant only |
| `limits_concurrency` | Solid Queue adapter | Solid Queue only |
| `perform_all_later` | Rails 7.1+ | Version-dependent |
| Recurring jobs format | Check `config/recurring.yml` vs `sidekiq_schedule.yml` | Respect existing choice |

```bash
# Detection commands
grep -r "config.active_job.queue_adapter" config/
grep -r "Current.account" app/
grep "^  rails " Gemfile.lock
ls config/recurring.yml config/sidekiq_schedule.yml 2>/dev/null
```
