If you have a queue configured as a [FIFO Queue](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html), Shoryuken will auto detect it, and just work ✨ 

## How does that work?

If you send a message using [`perform_async`](https://github.com/ruby-shoryuken/shoryuken/wiki/Sending-a-message) or [ActiveJob](https://github.com/ruby-shoryuken/shoryuken/wiki/Rails-Integration-Active-Job), Shoryuken will automatically set the Message Group ID to `ShoryukenMessage` and the Message Deduplication ID to a SHA-256 hash based on the message body.

If you want to set specific values to Message Group ID and Message Deduplication ID, 
you should use `send_message` instead:

```ruby
MyWorker.perform_async(
  'body',
  message_group_id: 'id',
  message_deduplication_id: 'id'
)

Shoryuken::Client.queues('queue.fifo').send_message(
  message_body: 'body', 
  message_group_id: 'id', 
  message_deduplication_id: 'id'
)
```

### Turning off automatic deduplication IDs

Two sends of an identical body within the SQS 5-minute deduplication window get the same Message Deduplication ID, so SQS drops the second one. To stop Shoryuken from generating the ID:

```ruby
# perform_async, Queue#send_message and Queue#send_messages
Shoryuken.fifo_message_deduplication = false

# ActiveJob adapter
Shoryuken.active_job_fifo_message_deduplication = false
```

Both default to `true`. An explicit `message_deduplication_id` is always used. With the setting at `false`, give a `message_deduplication_id` yourself or enable `ContentBasedDeduplication` on the queue.

*Note:* If you are fetching from a FIFO queue and not using the batch option for workers, the fetcher will fetch a maximum of one message at a time, in order to ensure that the messages for a particular Message Group ID are executed in FIFO order ([see here](https://github.com/ruby-shoryuken/shoryuken/blob/17e9df8d80d86485c1b80ef4465350a8c40fa90d/lib/shoryuken/fetcher.rb#L62-L71)). It is better to keep `concurrency > 1` in this case, so that your worker can service jobs from other Message Group IDs at the same time.

You might also consider using the `batch` option to fetch more messages at once, but process them in sequence. SQS will not release the next message for a particular Message Group ID unless the currently-received ones have been deleted. [See the AWS docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html#FIFO-queues-understanding-logic) (section entitled "Retrying multiple times") for more info.

## Receiving messages

There's no special configuration needed to receive FIFO messages, you only need to configure the queue in the [same way you would if it was a standard queue](https://github.com/ruby-shoryuken/shoryuken/wiki/Shoryuken-options#queues).

### Process one by one

If you want to receive and process messages one by one for a Message Group, configure `max_number_of_messages: 1 
` as follows:

```ruby
Shoryuken.sqs_client_receive_message_opts = { 
  max_number_of_messages: 1 
}
```

See [[Receive Message options]].

## Failures and dead-letter queues

When a worker raises an error, Shoryuken does not delete the message (see [[Retrying a message]]), unless the error matches the worker's [non_retryable_exceptions](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#non_retryable_exceptions). SQS returns no other message of the same Message Group ID until that message is deleted or becomes visible again ([see the AWS docs](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-understanding-logic.html)). What happens next depends on the [redrive policy](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) of the queue:

- **Without a redrive policy**, SQS delivers the failed message again after each visibility timeout. The message group stays blocked until the message succeeds, or until it is older than the [`MessageRetentionPeriod`](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_SetQueueAttributes.html) of the queue. Then SQS deletes the message, and the next message of the group runs.
- **With a redrive policy**, SQS moves the message to the dead-letter queue (DLQ) after `maxReceiveCount` receives. Then the next message of the same group runs, so the group is no longer processed in order. AWS advises not to use a DLQ with a FIFO queue if you need the exact order of messages.

For example, with `maxReceiveCount: 3`, a `paid` message fails 3 times and SQS moves it to the DLQ. Then the `shipped` message of the same group runs.

When you move the message back from the DLQ later, for example with `shoryuken sqs mv`, it is sent as a new message with the same Message Group ID (see [[Shoryuken SQS CLI]]). It goes to the end of its group: the redriven `paid` message runs after `shipped`.

If the order of a group matters, use one of these patterns:

- Keep the redrive policy, and make the worker check that the previous step is done before it does its work. If it is not done, the worker raises an error, so SQS delivers the message again. This message also moves to the DLQ after `maxReceiveCount` receives.
- Do not set a redrive policy. The group stays blocked until the message succeeds, so alert on the failures and fix the cause before the `MessageRetentionPeriod` ends. Else SQS deletes the message and it is lost.

### Detecting the last receive

In a worker, `sqs_msg.last_receive?` tells you whether this is the last receive before SQS moves the message to the DLQ. It is `true` when the `ApproximateReceiveCount` of the message has reached the `maxReceiveCount` of the queue. If the worker does not delete the message now, the next receive moves it to the DLQ, and the next message of the group runs. This is the point to alert:

```ruby
class OrderWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'orders.fifo', auto_delete: true

  def perform(sqs_msg, body)
    process(body)
  rescue StandardError
    notify_ops(sqs_msg.message_id) if sqs_msg.last_receive?
    raise
  end
end
```

`last_receive?` returns `false` when the queue has no redrive policy or the receive count is unknown. With the [[Shoryuken Inline adapter]] it always returns `false`.

`Shoryuken::Queue#max_receive_count` returns the `maxReceiveCount` of the redrive policy as an Integer, or `nil` when the queue has no redrive policy. Shoryuken reads it once per queue and caches it, so `last_receive?` does not make an SQS call for each message.
