# Bulk Enqueuing

Shoryuken provides efficient bulk enqueuing of ActiveJob jobs using the SQS `SendMessageBatch` API.

**Requires:** Rails 7.1+

## Overview

Instead of sending individual `SendMessage` calls for each job, bulk enqueuing batches jobs into groups of 10 and sends them with a single API call. This is significantly more efficient for enqueuing many jobs at once.

## Basic Usage

```ruby
# Create job instances
jobs = users.map { |user| NotifyUserJob.new(user) }

# Enqueue all jobs efficiently
ActiveJob.perform_all_later(jobs)
```

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                    Bulk Enqueue Flow                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  100 jobs → Group by queue → Batch into 10s → SendBatch     │
│                                                              │
│  Queue A (60 jobs)  → 6 × SendMessageBatch calls             │
│  Queue B (40 jobs)  → 4 × SendMessageBatch calls             │
│                                                              │
│  Total: 10 API calls instead of 100                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Processing Logic

1. Jobs are grouped by queue name
2. Each queue's jobs are batched into groups of 10 (SQS limit)
3. Each batch is sent with a single `send_message_batch` call
4. Success/failure is tracked per job

## Batch Limits

| Limit | Value |
|-------|-------|
| Messages per batch | 10 |
| Total batch size | 1 MB |
| Individual message size | 256 KB |

If a batch exceeds 1 MB, you may need to reduce the payload size of individual jobs.

## Examples

### Basic Bulk Enqueue

```ruby
# Send welcome emails to new users
new_users = User.where(created_at: 24.hours.ago..)
jobs = new_users.map { |user| WelcomeEmailJob.new(user) }

ActiveJob.perform_all_later(jobs)
```

### Multi-Queue Bulk Enqueue

Jobs targeting different queues are automatically grouped:

```ruby
jobs = []

# High priority notifications
urgent_users.each do |user|
  jobs << NotifyJob.new(user).tap { |j| j.queue_name = 'high_priority' }
end

# Normal priority notifications
regular_users.each do |user|
  jobs << NotifyJob.new(user).tap { |j| j.queue_name = 'default' }
end

# Both queues are batched efficiently
ActiveJob.perform_all_later(jobs)
```

### With Job Options

```ruby
jobs = orders.map do |order|
  ProcessOrderJob.new(order).tap do |job|
    # Set FIFO queue options
    job.message_group_id = order.customer_id.to_s
    job.message_deduplication_id = "order-#{order.id}"
  end
end

ActiveJob.perform_all_later(jobs)
```

### Checking Results

After bulk enqueuing, check which jobs were successfully enqueued:

```ruby
jobs = users.map { |user| NotifyJob.new(user) }
ActiveJob.perform_all_later(jobs)

# Check results
successful = jobs.select(&:successfully_enqueued?)
failed = jobs.reject(&:successfully_enqueued?)

if failed.any?
  Rails.logger.warn "Failed to enqueue #{failed.count} jobs"
  # Handle failures (retry, alert, etc.)
end
```

## Performance Comparison

| Method | 1000 Jobs | API Calls | Typical Time |
|--------|-----------|-----------|--------------|
| Individual `perform_later` | 1000 | 1000 | ~30-60s |
| Bulk `perform_all_later` | 1000 | 100 | ~3-6s |

**~10x faster** for bulk operations.

## Error Handling

### Partial Failures

SQS batch operations can partially succeed. Some jobs may be enqueued while others fail:

```ruby
jobs = users.map { |user| ProcessJob.new(user) }
enqueued_count = ActiveJob.perform_all_later(jobs)

if enqueued_count < jobs.count
  failed_jobs = jobs.reject(&:successfully_enqueued?)

  failed_jobs.each do |job|
    # Retry individually or log for investigation
    Rails.logger.error "Failed to enqueue job for #{job.arguments}"
  end
end
```

### Handling Large Payloads

If jobs have large payloads, batches may exceed the 1 MB limit:

```ruby
# For large payloads, consider storing data externally
class LargeDataJob < ApplicationJob
  def perform(data_key)
    data = Rails.cache.fetch(data_key)
    process(data)
  end
end

# Store data in cache, pass only the key
jobs = large_datasets.map do |data|
  key = "job_data:#{SecureRandom.uuid}"
  Rails.cache.write(key, data, expires_in: 1.hour)
  LargeDataJob.new(key)
end

ActiveJob.perform_all_later(jobs)
```

## Best Practices

### 1. Use for Batch Operations

Bulk enqueuing is ideal for:
- Batch notifications
- Data migrations
- Report generation
- Scheduled bulk operations

```ruby
# Good use case: Nightly cleanup
class NightlyCleanupJob < ApplicationJob
  def self.schedule_all
    jobs = Tenant.active.map { |t| new(t) }
    ActiveJob.perform_all_later(jobs)
  end
end
```

### 2. Handle Memory Efficiently

For very large batches, process in chunks to avoid memory issues:

```ruby
User.where(needs_notification: true).find_in_batches(batch_size: 500) do |users|
  jobs = users.map { |user| NotifyJob.new(user) }
  ActiveJob.perform_all_later(jobs)
end
```

### 3. Consider Queue Isolation

For critical jobs, consider separate queues:

```ruby
# Critical payment jobs get their own queue
payment_jobs = pending_payments.map do |p|
  ProcessPaymentJob.new(p).tap { |j| j.queue_name = 'payments' }
end

# Non-critical jobs on default queue
notification_jobs = users.map do |u|
  NotifyJob.new(u).tap { |j| j.queue_name = 'default' }
end

ActiveJob.perform_all_later(payment_jobs + notification_jobs)
```

## Implementation Details

The `enqueue_all` method in Shoryuken's adapter:

1. Groups jobs by `queue_name`
2. For each queue, processes jobs in batches of 10
3. Calls `send_message_batch` for each batch
4. Tracks success/failure per job via `successfully_enqueued?`
5. Returns total count of successfully enqueued jobs

```ruby
# Simplified implementation
def enqueue_all(jobs)
  jobs.group_by(&:queue_name).each do |queue_name, queue_jobs|
    queue = Shoryuken::Client.queues(queue_name)

    queue_jobs.each_slice(10) do |batch|
      entries = batch.map.with_index do |job, idx|
        { id: idx.to_s }.merge(message(queue, job))
      end

      response = queue.send_messages(entries: entries)

      # Mark successful jobs
      response.successful.each do |r|
        batch[r.id.to_i].successfully_enqueued = true
      end
    end
  end

  jobs.count(&:successfully_enqueued?)
end
```

## Related

- [[Rails Integration Active Job]] - Basic ActiveJob setup
- [[Sending a message]] - Individual message sending
- [AWS SQS SendMessageBatch](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SendMessageBatch.html)
