# Worker Options

Configure Shoryuken workers using `shoryuken_options`. These options are only available for native Shoryuken workers, not ActiveJob jobs (see [[Rails Integration Active Job]]).

## Options Summary

| Option | Default | Description |
|--------|---------|-------------|
| `queue` | `'default'` | Queue name(s) to process |
| `auto_delete` | `false` | Delete message after successful processing |
| `batch` | `false` | Receive messages in batches |
| `body_parser` | `:text` | Parse message body before `perform` |
| `auto_visibility_timeout` | `false` | Auto-extend visibility timeout |
| `retry_intervals` | `nil` | Exponential backoff intervals |

---

## queue

Associates a worker with one or more queues.

### Single Queue

```ruby
class HelloWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'hello'

  def perform(sqs_msg, body)
    puts body
  end
end
```

### Dynamic Queue Name

Use a lambda for environment-specific queues:

```ruby
class HelloWorker
  include Shoryuken::Worker

  shoryuken_options queue: -> { "#{Rails.env}_hello" }

  def perform(sqs_msg, body)
    # ...
  end
end
```

### Multiple Queues

One worker can process messages from multiple queues:

```ruby
class MultiQueueWorker
  include Shoryuken::Worker

  shoryuken_options queue: %w[queue1 queue2 queue3]

  def perform(sqs_msg, body)
    # ...
  end
end
```

---

## auto_delete

**Default:** `false`

Automatically deletes messages after successful processing.

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default', auto_delete: true

  def perform(sqs_msg, body)
    # Message is deleted after this method returns without error
    process(body)
  end
end
```

### Manual Deletion

When `auto_delete: false`, you must delete messages manually:

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default', auto_delete: false

  def perform(sqs_msg, body)
    if process(body)
      sqs_msg.delete  # Explicit deletion
    end
    # If not deleted, message returns to queue after visibility timeout
  end
end
```

**Note:** ActiveJob jobs have `auto_delete: true` by default.

---

## batch

**Default:** `false`

Receive up to 10 messages at once for batch processing.

```ruby
class BatchWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'events', batch: true, auto_delete: true

  def perform(sqs_msgs, bodies)
    # sqs_msgs and bodies are arrays
    bodies.each do |body|
      process(body)
    end
  end
end
```

### Use Cases

- Bulk database inserts
- Batch API calls
- High-throughput, low-latency jobs

### Considerations

- If any message raises an exception, `auto_delete` won't run for any message
- Middleware receives arrays for `sqs_msg` and `body` parameters
- Not compatible with `auto_visibility_timeout` or `retry_intervals`

---

## body_parser

**Default:** `:text`

Parse message body before calling `perform`.

### Built-in Parsers

```ruby
# JSON parsing
shoryuken_options body_parser: :json

# Raw text (default)
shoryuken_options body_parser: :text
```

### Custom Parser (Lambda)

```ruby
shoryuken_options body_parser: ->(sqs_msg) {
  REXML::Document.new(sqs_msg.body)
}
```

### Custom Parser (Class)

Any class that responds to `.parse`:

```ruby
shoryuken_options body_parser: JSON
shoryuken_options body_parser: Oj
shoryuken_options body_parser: MyCustomParser
```

### Example

```ruby
class JsonWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'json_queue', body_parser: :json

  def perform(sqs_msg, body)
    # body is already a Hash/Array
    puts body['name']
  end
end
```

---

## auto_visibility_timeout

**Default:** `false`

Automatically extends visibility timeout during long-running jobs.

```ruby
class LongRunningWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'long_jobs', auto_visibility_timeout: true

  def perform(sqs_msg, body)
    # Visibility timeout is extended automatically
    long_running_operation  # Can take longer than queue's visibility timeout
  end
end
```

### How It Works

5 seconds before the visibility timeout expires, Shoryuken resets it to the original value. This repeats until the job completes.

### Best Practices

Prefer setting a generous visibility timeout on the queue instead:

> If your worker in the worst case takes 2 minutes to consume a message, set the visibility_timeout to at least 4 minutes.

**Not supported** with `batch: true`.

---

## retry_intervals

**Default:** `nil`

Implement exponential backoff for failed jobs.

### Array of Intervals

```ruby
class RetryWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default',
                    retry_intervals: [60, 300, 1800, 3600]
                    # 1 min, 5 min, 30 min, 1 hour

  def perform(sqs_msg, body)
    # On failure, next retry after interval based on attempt count
    unreliable_operation
  end
end
```

### Dynamic Intervals (Lambda)

```ruby
shoryuken_options retry_intervals: ->(attempts) {
  (attempts ** 2) * 60  # Exponential: 60, 240, 540, 960...
}
```

### How It Works

When a job fails:
1. Shoryuken changes the message's visibility timeout
2. Message becomes visible after the interval
3. Retry count is tracked via message attributes

### Limits

- Maximum visibility timeout: **12 hours** (SQS limit)
- Intervals exceeding 12 hours are capped automatically

**Not supported** with `batch: true`.

---

## Per-Worker Middleware

Add middleware specific to a worker:

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default'

  server_middleware do |chain|
    chain.add WorkerSpecificMiddleware
  end

  def perform(sqs_msg, body)
    # ...
  end
end
```

See [[Middleware]] for more details.

---

## Complete Example

```ruby
class OrderProcessor
  include Shoryuken::Worker

  shoryuken_options(
    queue: -> { "#{Rails.env}_orders" },
    auto_delete: true,
    body_parser: :json,
    retry_intervals: [60, 300, 900, 3600]
  )

  server_middleware do |chain|
    chain.add MetricsMiddleware
  end

  def perform(sqs_msg, body)
    order = Order.find(body['order_id'])
    order.process!
  rescue ActiveRecord::RecordNotFound
    # Don't retry - order doesn't exist
    Rails.logger.warn "Order #{body['order_id']} not found"
  end
end
```

---

## ActiveJob Equivalent

For ActiveJob jobs, configure via job class:

```ruby
class MyJob < ApplicationJob
  queue_as :default

  # Retry with exponential backoff (built into ActiveJob)
  retry_on StandardError, wait: :polynomially_longer, attempts: 5

  def perform(args)
    # ...
  end
end
```

See [[Rails Integration Active Job]] for ActiveJob-specific options.

---

## Related

- [[Shoryuken options]] - Global configuration
- [[Middleware]] - Middleware chain
- [[Rails Integration Active Job]] - ActiveJob integration
- [[Retrying a message]] - Retry strategies
