# Rails Integration - Active Job

Shoryuken provides full support for Rails [Active Job](https://guides.rubyonrails.org/active_job_basics.html), enabling you to write processor-agnostic jobs that work with any queue backend.

## Requirements

- Rails 7.2+
- Ruby 3.2+

## Basic Setup

### 1. Configure the Queue Adapter

```ruby
# config/application.rb
module YourApp
  class Application < Rails::Application
    config.active_job.queue_adapter = :shoryuken
  end
end
```

### 2. Create an ApplicationJob

```ruby
# app/jobs/application_job.rb
class ApplicationJob < ActiveJob::Base
  # Retry on common transient errors
  retry_on StandardError, wait: :polynomially_longer, attempts: 5

  # Discard jobs that reference deleted records
  discard_on ActiveJob::DeserializationError
end
```

### 3. Create a Job

```ruby
# app/jobs/process_photo_job.rb
class ProcessPhotoJob < ApplicationJob
  queue_as :default

  def perform(photo)
    photo.process_image!
  end
end
```

### 4. Create a Shoryuken Initializer

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  # Replace Rails logger so messages are logged to Shoryuken's log
  Rails.logger = Shoryuken::Logging.logger
  Rails.logger.level = Rails.application.config.log_level

  # Add server middleware
  # config.server_middleware do |chain|
  #   chain.add MyMiddleware
  # end
end
```

### 5. Start Shoryuken

```shell
bundle exec shoryuken -q default -R
```

---

## Queue Adapters

### Standard Adapter (Synchronous)

The default adapter sends messages synchronously:

```ruby
# config/application.rb
config.active_job.queue_adapter = :shoryuken
```

### Concurrent Send Adapter (Asynchronous)

For high-throughput scenarios, use the async adapter with optional success/error handlers:

```ruby
# config/initializers/shoryuken.rb
success_handler = ->(response, job, options) {
  Rails.logger.info "Enqueued #{job.class.name} successfully"
}

error_handler = ->(error, job, options) {
  Rails.logger.error "Failed to enqueue #{job.class.name}: #{error.message}"
  # Report to error tracking service
  Sentry.capture_exception(error)
}

Rails.application.config.active_job.queue_adapter =
  ActiveJob::QueueAdapters::ShoryukenConcurrentSendAdapter.new(success_handler, error_handler)
```

---

## Queue Configuration

### Queue Name Prefixing

Active Job supports [queue name prefixing](https://guides.rubyonrails.org/active_job_basics.html#queues). To have Shoryuken honor these prefixes:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.active_job_queue_name_prefixing = true
```

Example with prefixing enabled:

```ruby
# config/environments/production.rb
config.active_job.queue_name_prefix = "production"

# This job will use queue "production_reports"
class ReportJob < ApplicationJob
  queue_as :reports
end
```

### Dynamic Queue Names

```ruby
class TenantJob < ApplicationJob
  queue_as { "tenant_#{Current.tenant_id}" }

  def perform(data)
    # Process tenant-specific work
  end
end
```

---

## SQS Message Parameters

Use `.set` to customize SQS message parameters:

```ruby
MyJob.set(
  message_group_id: 'group-123',           # For FIFO queues
  message_deduplication_id: 'dedupe-456',  # For FIFO queues
  message_attributes: {
    'correlation_id' => {
      string_value: SecureRandom.uuid,
      data_type: 'String'
    }
  }
).perform_later(data)
```

---

## Scheduling and Delays

### Delayed Execution

```ruby
# Process in 5 minutes
MyJob.set(wait: 5.minutes).perform_later(data)

# Process at a specific time
MyJob.set(wait_until: Date.tomorrow.noon).perform_later(data)
```

**Note:** SQS limits delays to 15 minutes maximum. For longer delays, use a scheduling solution like EventBridge Scheduler.

---

## Bulk Enqueuing (Rails 7.1+)

Enqueue multiple jobs efficiently using the SQS batch API:

```ruby
# Enqueue many jobs in batches of 10
jobs = users.map { |user| NotifyUserJob.new(user) }
ActiveJob.perform_all_later(jobs)
```

This uses `send_message_batch` internally, which is more efficient than individual `send_message` calls.

**Batch Limits:**
- Maximum 10 messages per batch
- Maximum 1MB total batch size

See [[Bulk Enqueuing]] for more details.

---

## ActiveJob Continuations (Rails 8.1+)

Shoryuken supports ActiveJob Continuations for graceful interruption of long-running jobs:

```ruby
class DataExportJob < ApplicationJob
  def perform(export)
    export.records.find_each do |record|
      # Check if Shoryuken is shutting down
      if stopping?
        # Save progress and re-enqueue
        export.update!(last_processed_id: record.id)
        self.class.perform_later(export)
        return
      end

      process_record(record)
    end

    export.mark_completed!
  end
end
```

The `stopping?` method returns `true` when Shoryuken receives a shutdown signal, allowing jobs to checkpoint their progress.

See [[ActiveJob Continuations]] for more details.

---

## CurrentAttributes Persistence

Automatically flow Rails `ActiveSupport::CurrentAttributes` from enqueue to job execution:

```ruby
# config/initializers/shoryuken.rb
require 'shoryuken/active_job/current_attributes'
Shoryuken::ActiveJob::CurrentAttributes.persist('Current')
```

```ruby
# app/models/current.rb
class Current < ActiveSupport::CurrentAttributes
  attribute :user, :request_id, :tenant
end
```

```ruby
# In your controller
Current.user = current_user
Current.tenant = current_tenant

# The job will have access to Current.user and Current.tenant
AuditJob.perform_later(action: 'updated_profile')
```

See [[CurrentAttributes]] for more details.

---

## Transaction Safety (Rails 7.2+)

Jobs are automatically enqueued after the database transaction commits:

```ruby
User.transaction do
  user = User.create!(name: 'Ken')
  WelcomeEmailJob.perform_later(user)  # Enqueued AFTER transaction commits
end
```

This prevents jobs from processing before the record exists. Shoryuken enables this by implementing `enqueue_after_transaction_commit?`.

---

## Retrying Failed Jobs

### Using ActiveJob retry_on

```ruby
class ProcessPaymentJob < ApplicationJob
  retry_on PaymentGatewayError, wait: :polynomially_longer, attempts: 5
  discard_on InvalidPaymentError

  def perform(payment)
    PaymentGateway.process(payment)
  end
end
```

### Using Shoryuken's Exponential Backoff

For more control, use Shoryuken's `retry_intervals`:

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default',
                    auto_delete: true,
                    retry_intervals: [60, 300, 3600]  # 1min, 5min, 1hr

  def perform(sqs_msg, body)
    # Process message
  end
end
```

See [[Retrying a message]] for more details.

---

## Error Handling

### Deserialization Errors

```ruby
class MyJob < ApplicationJob
  # Discard jobs with serialization errors (e.g., deleted records)
  discard_on ActiveJob::DeserializationError

  def perform(record)
    record.process!
  end
end
```

### Custom Exception Handlers

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.on_exception do |ex, queue, sqs_msg|
    Sentry.capture_exception(ex, extra: {
      queue: queue,
      message_id: sqs_msg.message_id
    })
  end
end
```

---

## Next Steps

- [[ActiveJob Continuations]] - Handle graceful shutdowns
- [[CurrentAttributes]] - Persist request context
- [[Bulk Enqueuing]] - Efficient batch operations
- [[Middleware]] - Add cross-cutting concerns
- [[Testing]] - Test your jobs
