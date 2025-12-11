# Sending a Message

This guide covers different ways to send messages to SQS queues with Shoryuken.

## Shoryuken Workers

### perform_async

Send a message for immediate processing:

```ruby
MyWorker.perform_async('Pablo')
```

With a Hash (automatically converted to JSON):

```ruby
MyWorker.perform_async(user_id: 123, action: 'notify')
```

Override the queue:

```ruby
MyWorker.perform_async('Pablo', queue: 'important')
```

### perform_in / perform_at

Delay message delivery (up to 15 minutes - SQS limit):

```ruby
# Delay by seconds
MyWorker.perform_in(60, 'Pablo')  # 60 seconds

# Delay until specific time
MyWorker.perform_at(5.minutes.from_now, 'Pablo')
```

---

## ActiveJob

### perform_later

```ruby
MyJob.perform_later(user_id: 123)
```

### Delayed Jobs

```ruby
# Delay by duration
MyJob.set(wait: 5.minutes).perform_later(user_id: 123)

# Delay until specific time
MyJob.set(wait_until: 1.hour.from_now).perform_later(user_id: 123)
```

### Bulk Enqueuing (Rails 7.1+)

Enqueue multiple jobs efficiently:

```ruby
jobs = users.map { |user| NotifyUserJob.new(user.id) }
ActiveJob.perform_all_later(jobs)
```

See [[Bulk Enqueuing]] for details.

---

## Direct SQS Client

Use the SQS client directly for more control:

### Single Message

```ruby
Shoryuken::Client.queues('default').send_message('message body')
```

With options:

```ruby
Shoryuken::Client.queues('default').send_message(
  message_body: { user_id: 123 }.to_json,
  message_attributes: {
    'type' => {
      string_value: 'notification',
      data_type: 'String'
    }
  }
)
```

### Multiple Messages

```ruby
Shoryuken::Client.queues('default').send_messages([
  'message 1',
  'message 2',
  'message 3'
])
```

With full options:

```ruby
Shoryuken::Client.queues('default').send_messages(
  entries: [
    { id: '1', message_body: 'msg 1' },
    { id: '2', message_body: 'msg 2' },
    { id: '3', message_body: 'msg 3', delay_seconds: 60 }
  ]
)
```

### Batch Limits

| Limit | Value |
|-------|-------|
| Max messages per batch | 10 |
| Max total batch size | 1 MB (256 KB × 4) |
| Max single message | 256 KB |

---

## Sending ActiveJob Messages Externally

Send messages from outside Rails (e.g., Lambda, external service):

```ruby
queue_name = 'my-queue'

job_args = {
  job_class: 'NotifyUserJob',
  job_id: SecureRandom.uuid,
  queue_name: queue_name,
  arguments: [
    {
      user_id: 123,
      message: 'Hello',
      _aj_symbol_keys: ['user_id', 'message']
    }
  ],
  executions: 0,
  locale: 'en'
}

message = {
  message_body: job_args.to_json,
  message_attributes: {
    shoryuken_class: {
      string_value: 'ActiveJob::QueueAdapters::ShoryukenAdapter::JobWrapper',
      data_type: 'String'
    }
  }
}

Shoryuken::Client.queues(queue_name).send_message(message)
```

---

## FIFO Queues

FIFO queues require message group ID and optionally deduplication ID:

```ruby
Shoryuken::Client.queues('my-queue.fifo').send_message(
  message_body: 'order processing',
  message_group_id: 'order-123',
  message_deduplication_id: SecureRandom.uuid
)
```

With content-based deduplication enabled on the queue, you can omit `message_deduplication_id`.

See [[FIFO Queues]] for more details.

---

## Message Attributes

Add metadata to messages:

```ruby
Shoryuken::Client.queues('default').send_message(
  message_body: 'test',
  message_attributes: {
    'correlation_id' => {
      string_value: SecureRandom.uuid,
      data_type: 'String'
    },
    'priority' => {
      string_value: '1',
      data_type: 'Number'
    },
    'binary_data' => {
      binary_value: Base64.encode64('raw data'),
      data_type: 'Binary'
    }
  }
)
```

Access in worker:

```ruby
def perform(sqs_msg, body)
  correlation_id = sqs_msg.message_attributes.dig('correlation_id', 'string_value')
end
```

---

## Concurrent Sending (Async)

Use the concurrent adapter for non-blocking sends:

```ruby
# config/application.rb
config.active_job.queue_adapter = ActiveJob::QueueAdapters::ShoryukenConcurrentSendAdapter.new(
  ->(job, response) { Rails.logger.info "Enqueued: #{job.job_id}" },
  ->(job, error) { Rails.logger.error "Failed: #{error.message}" }
)
```

See [[Rails Integration Active Job]] for details.

---

## Error Handling

### Rescue Send Errors

```ruby
begin
  MyWorker.perform_async('test')
rescue Aws::SQS::Errors::ServiceError => e
  Rails.logger.error "SQS error: #{e.message}"
  # Handle error (retry, fallback, etc.)
end
```

### Batch Failures

When sending batches, check for partial failures:

```ruby
response = Shoryuken::Client.queues('default').send_messages(
  entries: messages
)

if response.failed.any?
  response.failed.each do |failure|
    Rails.logger.error "Failed to send #{failure.id}: #{failure.message}"
  end
end
```

---

## AWS Credentials

Ensure AWS credentials are configured before sending messages. See [[Configure the AWS Client]].

---

## Related

- [[Bulk Enqueuing]] - Efficient batch enqueuing
- [[FIFO Queues]] - Ordered message processing
- [[Rails Integration Active Job]] - ActiveJob integration
- [[Worker options]] - Worker configuration
