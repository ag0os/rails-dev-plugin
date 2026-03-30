---
name: rails-jobs-patterns
description: Analyzes and recommends ActiveJob background processing patterns including idempotency, retry strategies, batch processing, scheduled jobs, and queue management. Use when creating background jobs, configuring Sidekiq, handling async workflows, or optimizing job performance. NOT for synchronous controller actions, model callbacks, service object design, or real-time Turbo Streams.
allowed-tools: Read, Grep, Glob
---

# Rails Background Job Patterns

Analyze and recommend patterns for reliable, efficient background jobs in Rails applications.

## Quick Reference

| Pattern | Use When | Key Method |
|---------|----------|------------|
| `retry_on` | Transient errors (network, timeout) | `retry_on Net::Error, wait: :exponentially_longer` |
| `discard_on` | Permanent failures (deleted record) | `discard_on ActiveJob::DeserializationError` |
| Idempotency guard | Job may run more than once | `return if already_processed?` |
| `with_lock` | Prevent concurrent execution | `record.with_lock { ... }` |
| Batch processing | Large datasets | `find_in_batches(batch_size: 100)` |
| Scheduled jobs | Recurring tasks | Solid Queue recurring or cron |

## Supporting Documentation

- [patterns.md](patterns.md) - Complete job patterns with detailed examples

## Core Principles

1. **Idempotency**: Jobs MUST be safe to run multiple times
2. **Pass IDs, not objects**: Avoid serialization issues
3. **Small payloads**: Keep arguments minimal and serializable
4. **Explicit error handling**: Use `retry_on` / `discard_on` declaratively
5. **Queue segmentation**: Separate urgent, default, and low-priority work

## Job Structure Template

```ruby
class ProcessOrderJob < ApplicationJob
  queue_as :default

  retry_on ActiveRecord::RecordNotFound, wait: 5.seconds, attempts: 3
  retry_on Net::OpenTimeout, wait: :exponentially_longer, attempts: 5
  discard_on ActiveJob::DeserializationError

  def perform(order_id)
    order = Order.find(order_id)
    return if order.processed?

    order.with_lock do
      return if order.processed?
      OrderProcessor.new(order).process!
      order.update!(status: "processed")
    end
  end
end
```

## Idempotency Patterns

```ruby
# Guard clause + lock
def perform(import_id)
  import = Import.find(import_id)
  return if import.completed?

  import.with_lock do
    return if import.completed?
    process_import(import)
    import.update!(status: "completed")
  end
end

# Unique constraint (database-level)
def perform(user_id, report_date)
  Report.create_or_find_by!(user_id: user_id, date: report_date) do |report|
    report.data = generate_report(user_id, report_date)
  end
end
```

## Error Handling Strategies

```ruby
class SendEmailJob < ApplicationJob
  # Transient: retry with backoff
  retry_on Net::SMTPServerError, wait: :exponentially_longer, attempts: 5

  # Transient: retry with fixed wait
  retry_on Timeout::Error, wait: 1.minute, attempts: 3

  # Permanent: discard with logging
  discard_on ActiveJob::DeserializationError do |job, error|
    Rails.logger.error("[#{job.class}] Discarded: #{error.message}")
  end

  # Domain-specific: handle inline
  def perform(payment_id)
    payment = Payment.find(payment_id)
    PaymentProcessor.charge!(payment)
  rescue PaymentProcessor::CardExpired
    payment.update!(status: "card_expired")
    # Do not retry - user action needed
  end
end
```

## Batch Processing

```ruby
class BatchExportJob < ApplicationJob
  def perform(export_id)
    export = Export.find(export_id)

    export.records.find_in_batches(batch_size: 500) do |batch|
      batch.each { |record| ProcessRecordJob.perform_later(record.id) }
      export.increment!(:processed_count, batch.size)
    end

    export.update!(status: "completed")
  end
end

# Self-splitting for very large datasets
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

## Scheduled Jobs

```ruby
# Recurring job with duplicate prevention
class DailyReportJob < ApplicationJob
  def perform(date = Date.current)
    return if Report.exists?(date: date, report_type: "daily")

    report = Report.create!(date: date, report_type: "daily", data: generate_data(date))
    ReportMailer.daily(report).deliver_later
  end
end
```

## Queue Configuration

```yaml
# config/sidekiq.yml
:queues:
  - [critical, 6]  # payments, auth
  - [default, 3]   # standard processing
  - [low, 1]       # reports, cleanup
  - [mailers, 2]   # email delivery
```

## Monitoring and Logging

```ruby
class MonitoredJob < ApplicationJob
  around_perform do |job, block|
    start = Time.current
    Rails.logger.info("[#{job.class}] Starting args=#{job.arguments}")
    block.call
    Rails.logger.info("[#{job.class}] Completed in #{Time.current - start}s")
  rescue => e
    Rails.logger.error("[#{job.class}] Failed: #{e.message}")
    raise
  end
end
```

## Anti-Patterns

| Anti-Pattern | Problem | Fix |
|-------------|---------|-----|
| Passing full objects as args | Serialization failures, stale data | Pass IDs, reload inside job |
| No idempotency guard | Duplicate processing on retry | Check status before processing |
| Bare `rescue => e` swallowing errors | Silent failures, no retries | Re-raise after logging |
| Long-running single job (>30 min) | Memory bloat, queue blocking | Split into batches |
| Business logic inside job | Hard to test, tight coupling | **Omakase:** delegate to model methods. **Service-oriented:** delegate to service objects |
| No queue segmentation | Urgent jobs blocked by bulk work | Use priority queues |
| Synchronous external calls in jobs | Timeouts block workers | Set timeouts, use circuit breakers |

## Output Format

When analyzing or creating jobs, provide:
1. **Job file** in `app/jobs/` with retry/discard configuration
2. **Idempotency strategy** (guard clause, lock, unique constraint)
3. **Queue assignment** with rationale
4. **Test outline** using `ActiveJob::TestHelper`
5. **Monitoring** notes (logging, metrics, alerting)
