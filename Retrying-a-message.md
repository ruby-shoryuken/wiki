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

The example above will cause the message to be available again in case the `do_something` raises an exception. When [auto_delete](https://github.com/phstc/shoryuken/wiki/Worker-options#auto_delete) is set to `true`, Shoryuken will auto delete the message in case of the worker doesn't raise an exception.

Check the [Dead Letter Queue documentation](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) to prevent the message being processed _forever_.

## Exponential backoff 

Shoryuken also supports exponential backoff, have a look at [retry_intervals](https://github.com/phstc/shoryuken/wiki/Worker-options#retry_intervals) for more details.