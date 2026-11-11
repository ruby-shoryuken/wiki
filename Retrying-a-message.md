Good news - you don't need to do anything to retry a message, SQS will handle that automatically for you. Basically do not call `sqs_msg.delete` if you want to retry a message. The message will become available again when it reaches the [visibility_timeout](http://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html).

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'default', auto_delete: true

  def perform(sqs_msg, body)
    do_something(sqs_msg, body) # raises an exception
  end
end
```

The example above will cause the message to be available again in case the `do_something` raises an exception. When [auto_delete](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#auto_delete) is set to `true`, `auto_delete` deletes the message only when the worker does not raise an exception. If a server middleware added with `chain.add` rescues the exception and does not raise it again, `auto_delete` deletes the message too (see [[Middleware]]).

Check the [Dead Letter Queue documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) to prevent the message being processed _forever_. On a FIFO queue, a dead-letter queue breaks the order of a message group, see [FIFO Queues](https://github.com/ruby-shoryuken/shoryuken/wiki/FIFO-Queues#failures-and-dead-letter-queues). To find out in the worker that the message moves to the DLQ on its next receive, use `sqs_msg.last_receive?`, see [Detecting the last receive](https://github.com/ruby-shoryuken/shoryuken/wiki/FIFO-Queues#detecting-the-last-receive).

To make the message visible again sooner, set a shorter visibility timeout before you raise:

```ruby
sqs_msg.visibility_timeout = 0 # visible again at once
```

## Exponential backoff

Shoryuken also supports exponential backoff. See [retry_intervals](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#retry_intervals).

To delete a message instead of retrying it for some exceptions, see [non_retryable_exceptions](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#non_retryable_exceptions).

## Retrying a message by hand

SQS has no call that retries one message. To run a failed message again by hand, for example one that your middleware stored, send its body and message attributes again as a new message. See [[Sending a message]].

```ruby
Shoryuken::Client.queues('my-queue').send_message(
  message_body: stored_body,
  message_attributes: stored_message_attributes
)
```

- **Message attributes:** Shoryuken picks the worker from the `shoryuken_class` message attribute. Without it, Shoryuken uses the worker registered for the queue. `sqs_msg.message_attributes` holds structs: store `sqs_msg.message_attributes.transform_values(&:to_h)`. Give each attribute value to `send_message` as a Hash with symbol keys (`data_type` and `string_value` or `binary_value`). After a JSON round trip the keys are strings, and the send fails with `missing required parameter`.
- **The original message:** what happens to it depends on your middleware. If your middleware raises the error again, the message stays in the queue, and SQS delivers it again when its visibility timeout ends. Only an error that matches the worker's [non_retryable_exceptions](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#non_retryable_exceptions) is different: Shoryuken deletes that message. If your middleware (added with `chain.add`, see [[Middleware]]) rescues the error and does not raise it again, `auto_delete: true` deletes the message. When the message stays in the queue, both it and the new message run. To retry it only by hand, call `sqs_msg.delete` after you store it.
- **FIFO queues:** pass the original `message_group_id` (`sqs_msg.attributes['MessageGroupId']`); else Shoryuken sends the message to the group `ShoryukenMessage`. Pass a new `message_deduplication_id`, for example `SecureRandom.uuid`. By default, Shoryuken sets it to a SHA-256 hash of the body, and SQS drops a message whose deduplication ID it saw in the last 5 minutes. See [[FIFO Queues]].

To send the messages of a dead-letter queue back to their queue, use `shoryuken sqs mv DLQ-NAME QUEUE-NAME`, or `shoryuken sqs dump DLQ-NAME` and then `shoryuken sqs requeue QUEUE-NAME PATH`. See [[Shoryuken SQS CLI]].
