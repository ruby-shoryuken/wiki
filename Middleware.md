# Middleware

Middleware provides a way to add cross-cutting concerns to message processing. Each message passes through the middleware chain before reaching the worker.

## Built-in Middleware

Shoryuken includes several built-in server middleware:

| Middleware | Purpose | Auto-enabled |
|------------|---------|--------------|
| `ActiveRecord` | Clears database connections after each job | No |
| `AutoDelete` | Deletes messages after successful processing | Via `auto_delete` option |
| `AutoExtendVisibility` | Extends visibility timeout during processing | Via `auto_visibility_timeout` option |
| `ExponentialBackoffRetry` | Implements retry with exponential backoff | Via `retry_intervals` option |
| `Timing` | Logs processing duration | No |

---

## Creating Middleware

### Basic Structure

```ruby
class MyMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    # Before processing
    puts "Processing message #{sqs_msg.message_id}"

    yield  # Continue to next middleware/worker

    # After processing (only on success)
    puts "Completed message #{sqs_msg.message_id}"
  rescue => e
    # Handle errors
    puts "Failed: #{e.message}"
    raise
  end
end
```

### With Initialization Parameters

```ruby
class LoggingMiddleware
  def initialize(logger:)
    @logger = logger
  end

  def call(worker_instance, queue, sqs_msg, body)
    @logger.info "Starting #{worker_instance.class.name}"
    yield
    @logger.info "Completed #{worker_instance.class.name}"
  end
end

# Registration
Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add LoggingMiddleware, logger: Rails.logger
  end
end
```

---

## Registering Middleware

### Global Middleware

```ruby
# config/initializers/shoryuken.rb
Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add MyMiddleware
  end
end
```

### Chain Operations

```ruby
config.server_middleware do |chain|
  # Add to end of chain
  chain.add MyMiddleware

  # Add with parameters
  chain.add MyMiddleware, foo: 1, bar: 2

  # Insert before another middleware
  chain.insert_before ExistingMiddleware, NewMiddleware

  # Insert after another middleware
  chain.insert_after ExistingMiddleware, NewMiddleware

  # Remove middleware
  chain.remove UnwantedMiddleware

  # Clear all middleware
  chain.clear
end
```

### Per-Worker Middleware

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default'

  # Add worker-specific middleware
  server_middleware do |chain|
    chain.add WorkerSpecificMiddleware
  end

  def perform(sqs_msg, body)
    # ...
  end
end
```

### Replacing Global Middleware for a Worker

```ruby
class MyWorker
  include Shoryuken::Worker

  server_middleware do |chain|
    # Clear all global middleware
    chain.clear

    # Add only what this worker needs
    chain.add MinimalMiddleware
  end

  def perform(sqs_msg, body)
    # ...
  end
end
```

---

## Built-in Middleware Details

### ActiveRecord Middleware

Ensures database connections are returned to the pool after each job:

```ruby
# Enable manually if needed
Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add Shoryuken::Middleware::Server::ActiveRecord
  end
end
```

**Note:** For Rails applications, ActiveRecord connections are typically managed automatically. Add this middleware if you encounter connection pool exhaustion.

### AutoDelete Middleware

Automatically deletes messages after successful processing. Enabled via worker options:

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default', auto_delete: true

  def perform(sqs_msg, body)
    # Message is deleted after this returns successfully
  end
end
```

### AutoExtendVisibility Middleware

Extends the message visibility timeout during long-running jobs:

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default', auto_visibility_timeout: true

  def perform(sqs_msg, body)
    # Visibility timeout is extended automatically during processing
    long_running_operation
  end
end
```

### ExponentialBackoffRetry Middleware

Implements retry with exponential backoff:

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default',
                    retry_intervals: [60, 300, 3600]  # 1min, 5min, 1hr

  def perform(sqs_msg, body)
    # On failure, message visibility timeout is set based on retry count
  end
end
```

### Timing Middleware

Logs the duration of message processing:

```ruby
Shoryuken.configure_server do |config|
  config.server_middleware do |chain|
    chain.add Shoryuken::Middleware::Server::Timing
  end
end
```

---

## Common Middleware Examples

### Error Tracking (Sentry)

```ruby
class SentryMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    yield
  rescue => e
    Sentry.capture_exception(e, extra: {
      queue: queue,
      message_id: sqs_msg.message_id,
      worker: worker_instance.class.name
    })
    raise
  end
end
```

### Request ID Tracking

```ruby
class RequestIdMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    request_id = sqs_msg.message_attributes.dig('request_id', 'string_value') ||
                 SecureRandom.uuid

    Thread.current[:request_id] = request_id

    Rails.logger.tagged(request_id) do
      yield
    end
  ensure
    Thread.current[:request_id] = nil
  end
end
```

### Metrics Collection

```ruby
class MetricsMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    start_time = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    success = false

    yield
    success = true
  rescue => e
    StatsD.increment("shoryuken.job.failed", tags: ["queue:#{queue}"])
    raise
  ensure
    duration = Process.clock_gettime(Process::CLOCK_MONOTONIC) - start_time

    StatsD.timing("shoryuken.job.duration", duration * 1000, tags: [
      "queue:#{queue}",
      "worker:#{worker_instance.class.name}",
      "success:#{success}"
    ])

    StatsD.increment("shoryuken.job.processed", tags: ["queue:#{queue}"]) if success
  end
end
```

### Rate Limiting

```ruby
class RateLimitMiddleware
  def initialize(redis:, limit:, period:)
    @redis = redis
    @limit = limit
    @period = period
  end

  def call(worker_instance, queue, sqs_msg, body)
    key = "rate_limit:#{worker_instance.class.name}"

    if @redis.incr(key) > @limit
      # Re-queue for later processing
      raise "Rate limit exceeded"
    end

    @redis.expire(key, @period)
    yield
  end
end
```

---

## Rejecting Messages

To reject a message without processing, don't call `yield`:

```ruby
class ValidationMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    if valid?(body)
      yield
    else
      Shoryuken.logger.warn "Rejecting invalid message #{sqs_msg.message_id}"
      sqs_msg.delete  # Remove from queue
      # Don't yield - message won't be processed
    end
  end

  private

  def valid?(body)
    body.is_a?(Hash) && body['type'].present?
  end
end
```

---

## Batch Processing

When batch processing is enabled, `sqs_msg` and `body` are arrays:

```ruby
class BatchMiddleware
  def call(worker_instance, queue, sqs_msg, body)
    if sqs_msg.is_a?(Array)
      # Batch mode
      Shoryuken.logger.info "Processing batch of #{sqs_msg.size} messages"
    else
      # Single message mode
      Shoryuken.logger.info "Processing message #{sqs_msg.message_id}"
    end

    yield
  end
end
```

---

## Middleware Order

Middleware executes in the order registered:

```ruby
config.server_middleware do |chain|
  chain.add FirstMiddleware   # Runs first (outer)
  chain.add SecondMiddleware  # Runs second
  chain.add ThirdMiddleware   # Runs last (inner)
end
```

Execution flow:

```
FirstMiddleware.before
  SecondMiddleware.before
    ThirdMiddleware.before
      Worker.perform
    ThirdMiddleware.after
  SecondMiddleware.after
FirstMiddleware.after
```

---

## Related

- [[Worker options]] - Worker-level configuration
- [[Sentry.io Integration]] - Error tracking example
- [[Lifecycle Events]] - Process-level hooks
