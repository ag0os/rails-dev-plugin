# Background Job Patterns

Detailed patterns for ActiveJob and Sidekiq in Rails applications.

## Job Structure

### Basic Job with Error Handling

```ruby
class ProcessOrderJob < ApplicationJob
  queue_as :default

  # Retry on transient errors
  retry_on ActiveRecord::RecordNotFound, wait: 5.seconds, attempts: 3
  retry_on Net::OpenTimeout, wait: :exponentially_longer, attempts: 5
  retry_on Timeout::Error, wait: 1.minute, attempts: 3

  # Don't retry on permanent failures
  discard_on ActiveJob::DeserializationError do |job, error|
    Rails.logger.error "Failed to deserialize job #{job.job_id}: #{error.message}"
  end

  def perform(order_id)
    order = Order.find(order_id)

    OrderProcessor.new(order).process!
    OrderMailer.confirmation(order).deliver_later
  rescue StandardError => e
    Rails.logger.error "Failed to process order #{order_id}: #{e.message}"
    raise  # Re-raise to trigger retry
  end
end
```

### Dynamic Queue Selection

```ruby
class NotificationJob < ApplicationJob
  queue_as do
    user = arguments.first
    user.premium? ? :high_priority : :default
  end

  def perform(user, message)
    NotificationService.new(user).send(message)
  end
end
```

## Idempotency Patterns

### Status Check Guard

```ruby
class ProcessPaymentJob < ApplicationJob
  def perform(payment_id)
    payment = Payment.find(payment_id)

    # Guard: already processed
    return if payment.completed?

    process_payment(payment)
  end
end
```

### Lock-Based Idempotency

```ruby
class ImportDataJob < ApplicationJob
  def perform(import_id)
    import = Import.find(import_id)

    import.with_lock do
      return if import.completed?

      process_import(import)
      import.update!(status: 'completed', completed_at: Time.current)
    end
  end
end
```

### Unique Job Keys (with Sidekiq Enterprise or sidekiq-unique-jobs)

```ruby
class SyncUserJob < ApplicationJob
  # Ensure only one job per user runs at a time
  sidekiq_options lock: :until_executed,
                  lock_args_method: ->(args) { [args.first] }

  def perform(user_id)
    UserSyncService.new(user_id).sync!
  end
end
```

## Batch Processing

### Find in Batches

```ruby
class BulkEmailJob < ApplicationJob
  BATCH_SIZE = 100

  def perform(campaign_id)
    campaign = Campaign.find(campaign_id)

    campaign.subscribers.find_in_batches(batch_size: BATCH_SIZE) do |batch|
      batch.each do |subscriber|
        SendCampaignEmailJob.perform_later(campaign_id, subscriber.id)
      end

      # Update progress
      campaign.increment!(:processed_count, batch.size)
    end
  end
end
```

### Self-Chaining Batch Job

```ruby
class LargeDataProcessJob < ApplicationJob
  BATCH_SIZE = 1000

  def perform(dataset_id, offset = 0)
    dataset = Dataset.find(dataset_id)
    batch = dataset.records.offset(offset).limit(BATCH_SIZE)

    return if batch.empty?

    process_batch(batch)

    # Queue next batch
    self.class.perform_later(dataset_id, offset + BATCH_SIZE)
  end

  private

  def process_batch(records)
    records.each do |record|
      process_record(record)
    end
  end
end
```

### Parallel Batch Processing

```ruby
class ParallelProcessJob < ApplicationJob
  def perform(ids)
    # Split into smaller batches and process in parallel
    ids.each_slice(100) do |batch_ids|
      ProcessBatchJob.perform_later(batch_ids)
    end
  end
end
```

## Scheduled Jobs

### Recurring Job Pattern

```ruby
class DailyReportJob < ApplicationJob
  def perform(date = Date.current)
    # Idempotency: check if already run
    return if Report.exists?(date: date, type: 'daily')

    report = Report.create!(
      date: date,
      type: 'daily',
      data: generate_report_data(date)
    )

    ReportMailer.daily_report(report).deliver_later
  end

  private

  def generate_report_data(date)
    {
      orders: Order.where(created_at: date.all_day).count,
      revenue: Order.where(created_at: date.all_day).sum(:total),
      new_users: User.where(created_at: date.all_day).count
    }
  end
end
```

### With sidekiq-cron or sidekiq-scheduler

```yaml
# config/sidekiq_schedule.yml
daily_report:
  cron: "0 2 * * *"  # 2 AM daily
  class: DailyReportJob
  queue: low

hourly_sync:
  every: "1h"
  class: SyncDataJob
  queue: default
```

## Error Handling Strategies

### Custom Error Types

```ruby
class PaymentError < StandardError; end
class InsufficientFundsError < PaymentError; end
class CardExpiredError < PaymentError; end

class ProcessPaymentJob < ApplicationJob
  # Retry billing errors
  retry_on PaymentError, wait: :exponentially_longer, attempts: 5

  # Don't retry card issues - user needs to update
  discard_on CardExpiredError do |job, error|
    payment = Payment.find(job.arguments.first)
    payment.update!(status: 'card_expired')
    PaymentMailer.card_expired(payment).deliver_later
  end

  def perform(payment_id)
    payment = Payment.find(payment_id)
    PaymentProcessor.charge!(payment)
  end
end
```

### Dead Letter Queue Pattern

```ruby
class ReliableJob < ApplicationJob
  retry_on StandardError, attempts: 5

  after_discard do |job, error|
    # Record failed job for manual review
    FailedJob.create!(
      job_class: job.class.name,
      arguments: job.arguments,
      error_class: error.class.name,
      error_message: error.message,
      failed_at: Time.current
    )

    # Alert team
    SlackNotifier.notify("Job failed: #{job.class.name} - #{error.message}")
  end
end
```

## Monitoring and Logging

### Instrumented Job

```ruby
class InstrumentedJob < ApplicationJob
  around_perform do |job, block|
    start_time = Time.current

    Rails.logger.info({
      event: 'job_started',
      job_class: job.class.name,
      job_id: job.job_id,
      arguments: job.arguments
    }.to_json)

    block.call

    duration = Time.current - start_time
    Rails.logger.info({
      event: 'job_completed',
      job_class: job.class.name,
      job_id: job.job_id,
      duration_ms: (duration * 1000).round
    }.to_json)

    # Track metrics
    StatsD.timing("jobs.#{job.class.name.underscore}.duration", duration)
    StatsD.increment("jobs.#{job.class.name.underscore}.success")
  rescue StandardError => e
    StatsD.increment("jobs.#{job.class.name.underscore}.failure")
    raise
  end
end
```

## Queue Configuration

### Sidekiq Configuration

```yaml
# config/sidekiq.yml
:concurrency: 10

:queues:
  - [critical, 6]    # 6x weight
  - [default, 3]     # 3x weight
  - [low, 1]         # 1x weight
  - [mailers, 2]     # For ActionMailer

# Production with memory limits
:max_retries: 25
:dead_max_jobs: 10000
```

### Queue-Specific Workers

```ruby
# Critical operations
class ChargePaymentJob < ApplicationJob
  queue_as :critical
end

# Default priority
class SendNotificationJob < ApplicationJob
  queue_as :default
end

# Background cleanup
class CleanupJob < ApplicationJob
  queue_as :low
end
```

## Testing Patterns

### Test Job Execution

```ruby
RSpec.describe ProcessOrderJob, type: :job do
  include ActiveJob::TestHelper

  describe '#perform' do
    let(:order) { create(:order, status: 'pending') }

    it 'processes the order' do
      expect {
        described_class.perform_now(order.id)
      }.to change { order.reload.status }.to('processed')
    end

    it 'sends confirmation email' do
      expect {
        described_class.perform_now(order.id)
      }.to have_enqueued_job(ActionMailer::MailDeliveryJob)
    end
  end

  describe 'error handling' do
    it 'retries on network errors' do
      allow(OrderProcessor).to receive(:new).and_raise(Net::OpenTimeout)

      assert_performed_jobs 0 do
        described_class.perform_later(create(:order).id)
      end
    end
  end
end
```

### Test Enqueuing

```ruby
it 'enqueues with correct arguments' do
  expect {
    ProcessOrderJob.perform_later(order.id)
  }.to have_enqueued_job(ProcessOrderJob)
    .with(order.id)
    .on_queue('default')
end
```

## Directory Organization

```
app/jobs/
├── application_job.rb
├── concerns/
│   ├── idempotent.rb
│   └── instrumented.rb
├── orders/
│   ├── process_order_job.rb
│   └── cancel_order_job.rb
├── payments/
│   └── process_payment_job.rb
├── notifications/
│   └── send_notification_job.rb
└── maintenance/
    ├── cleanup_job.rb
    └── daily_report_job.rb
```

## Best Practices Summary

### Do
- Make jobs idempotent (safe to retry)
- Pass IDs, not objects
- Use appropriate queues
- Handle errors explicitly
- Log job execution
- Test thoroughly

### Don't
- Store state in instance variables
- Perform synchronous operations that could be async
- Create jobs with large payloads
- Ignore failed jobs
- Skip idempotency guards
