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

The example above will cause the message to be available again in case the `do_something` raises an exception. When [auto_delete](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#auto_delete) is set to `true`, Shoryuken deletes the message only when the worker does not raise an exception.

Check the [Dead Letter Queue documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) to prevent the message being processed _forever_. On a FIFO queue, a dead-letter queue breaks the order of a message group, see [FIFO Queues](https://github.com/ruby-shoryuken/shoryuken/wiki/FIFO-Queues#failures-and-dead-letter-queues). To find out in the worker that the message moves to the DLQ on its next receive, use `sqs_msg.last_receive?`, see [Detecting the last receive](https://github.com/ruby-shoryuken/shoryuken/wiki/FIFO-Queues#detecting-the-last-receive).

To make the message visible again sooner, set a shorter visibility timeout before you raise:

```ruby
sqs_msg.visibility_timeout = 0 # visible again at once
```

## Exponential backoff

Shoryuken also supports exponential backoff. See [retry_intervals](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#retry_intervals).

To delete a message instead of retrying it for some exceptions, see [non_retryable_exceptions](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#non_retryable_exceptions).