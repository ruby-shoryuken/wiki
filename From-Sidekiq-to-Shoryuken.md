# From Sidekiq to Shoryuken

This guide helps you migrate from Sidekiq to Shoryuken. The transition is mostly straightforward since Shoryuken's API is similar to Sidekiq's.

## Migration Strategy

1. **Stop sending jobs to Sidekiq** - Update code to use Shoryuken
2. **Start Shoryuken workers** - Begin processing new jobs
3. **Keep Sidekiq running** - Until it drains pending jobs from Redis

---

## Feature Comparison

| Feature | Sidekiq | Shoryuken |
|---------|---------|-----------|
| Backend | Redis | AWS SQS |
| Web UI | Built-in | Not included |
| Retry mechanism | Built-in | Via `retry_intervals` or DLQ |
| Scheduled jobs | Built-in (Redis) | SQS delay (15 min max) |
| Unique jobs | Sidekiq Enterprise | Not built-in |
| Batches | Sidekiq Pro | Not built-in |
| Rate limiting | Sidekiq Enterprise | Not built-in |
| CurrentAttributes | Sidekiq 7+ | Shoryuken v7+ |

---

## Worker Migration

### Sidekiq Worker

```ruby
class MyWorker
  include Sidekiq::Worker

  sidekiq_options queue: 'my_queue', retry: 5

  def perform(user_id, action)
    user = User.find(user_id)
    user.send(action)
  end
end
```

### Shoryuken Worker

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'my_queue',
                    auto_delete: true,
                    body_parser: :json,
                    retry_intervals: [60, 300, 900, 3600]

  def perform(sqs_msg, body)
    user = User.find(body['user_id'])
    user.send(body['action'])
  end
end
```

### Key Differences

| Aspect | Sidekiq | Shoryuken |
|--------|---------|-----------|
| Module | `Sidekiq::Worker` | `Shoryuken::Worker` |
| Options | `sidekiq_options` | `shoryuken_options` |
| Perform args | Multiple args | `sqs_msg, body` |
| Auto delete | Automatic | `auto_delete: true` |
| Retries | `retry: N` | `retry_intervals: [...]` |

---

## Multiple Arguments

Sidekiq passes multiple arguments directly. Shoryuken passes the message and parsed body.

### Sidekiq

```ruby
def perform(user_id, post_id, action)
  # Direct access to args
end

MyWorker.perform_async(123, 456, 'publish')
```

### Shoryuken

```ruby
shoryuken_options body_parser: :json

def perform(sqs_msg, body)
  user_id = body['user_id']
  post_id = body['post_id']
  action = body['action']
end

MyWorker.perform_async(user_id: 123, post_id: 456, action: 'publish')
```

---

## ActiveJob (Recommended)

Using ActiveJob makes migration simpler - no worker changes needed:

### Sidekiq Configuration

```ruby
# config/application.rb
config.active_job.queue_adapter = :sidekiq
```

### Shoryuken Configuration

```ruby
# config/application.rb
config.active_job.queue_adapter = :shoryuken
```

### Job Class (No Changes)

```ruby
class MyJob < ApplicationJob
  queue_as :default

  def perform(user_id, action)
    # Same code works with both adapters
  end
end
```

---

## Configuration File

### sidekiq.yml

```yaml
concurrency: 25
pidfile: tmp/pids/sidekiq.pid
queues:
  - default
  - [critical, 3]
  - [low, 1]
```

### shoryuken.yml

```yaml
concurrency: 25
timeout: 25
pidfile: tmp/pids/shoryuken.pid
queues:
  - default
  - [critical, 3]
  - [low, 1]
```

**Note:** Add `timeout` for graceful shutdown behavior.

---

## Sending Jobs

### Sidekiq

```ruby
MyWorker.perform_async('test')
MyWorker.perform_in(5.minutes, 'test')
MyWorker.perform_at(1.hour.from_now, 'test')
```

### Shoryuken

```ruby
MyWorker.perform_async('test')
MyWorker.perform_in(5.minutes, 'test')  # Max 15 minutes (SQS limit)
MyWorker.perform_at(1.hour.from_now, 'test')  # Max 15 minutes delay
```

**Note:** SQS has a maximum delay of 15 minutes. For longer delays, use scheduled jobs with a scheduler or re-enqueue pattern.

---

## Running Workers

### Sidekiq

```bash
bundle exec sidekiq -C config/sidekiq.yml
```

### Shoryuken

```bash
bundle exec shoryuken -R -C config/shoryuken.yml
```

**Note:** Use `-R` flag for Rails applications.

---

## Retry Handling

### Sidekiq

```ruby
sidekiq_options retry: 5  # Automatic exponential backoff
```

### Shoryuken

```ruby
# Option 1: Explicit intervals
shoryuken_options retry_intervals: [60, 300, 900, 3600, 7200]

# Option 2: Dynamic calculation
shoryuken_options retry_intervals: ->(attempts) { (attempts ** 2) * 60 }

# Option 3: Use Dead Letter Queue (recommended for production)
# Configure in SQS console
```

---

## Middleware

### Sidekiq

```ruby
Sidekiq.configure_server do |config|
  config.server_middleware do |chain|
    chain.add MyMiddleware
  end
end
```

### Shoryuken

```ruby
Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add MyMiddleware
  end
end
```

Middleware signature differs slightly:

```ruby
# Sidekiq
def call(worker, job, queue)
  yield
end

# Shoryuken
def call(worker_instance, queue, sqs_msg, body)
  yield
end
```

---

## CurrentAttributes (Sidekiq 7+ / Shoryuken 7+)

### Sidekiq

```ruby
# Automatically persisted in Sidekiq 7+
class Current < ActiveSupport::CurrentAttributes
  attribute :user
end
```

### Shoryuken

```ruby
# config/initializers/shoryuken.rb
Shoryuken::ActiveJob::CurrentAttributes.persist('Current')
```

---

## Things to Consider

### No Web UI

Shoryuken doesn't include a web UI. Monitor via:
- AWS CloudWatch (SQS metrics)
- Custom dashboards
- [[Monitoring and Metrics]]

### No Built-in Scheduled Jobs

SQS delays max 15 minutes. For scheduled jobs:
- Use AWS EventBridge Scheduler
- Use a separate scheduler service
- Re-enqueue pattern for longer delays

### Queue Creation

Create SQS queues before deployment:

```bash
shoryuken sqs create my_queue
shoryuken sqs create critical
```

### IAM Permissions

Ensure your AWS credentials have SQS access. See [[Amazon SQS IAM Policy for Shoryuken]].

---

## Migration Checklist

- [ ] Create SQS queues for each Sidekiq queue
- [ ] Configure AWS credentials
- [ ] Update Gemfile (`sidekiq` → `shoryuken`, add `aws-sdk-sqs`)
- [ ] Convert workers or switch ActiveJob adapter
- [ ] Update configuration file
- [ ] Add `auto_delete: true` to workers
- [ ] Configure retry strategy (intervals or DLQ)
- [ ] Update deployment scripts
- [ ] Set up monitoring (CloudWatch, etc.)
- [ ] Test thoroughly in staging

---

## Related

- [[Getting Started]] - Initial setup
- [[Worker options]] - Worker configuration
- [[Shoryuken options]] - Global configuration
- [[Rails Integration Active Job]] - ActiveJob integration
- [[CurrentAttributes]] - Request context persistence
