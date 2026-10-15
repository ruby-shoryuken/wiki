*Note:* `shoryuken_options` is only available in Shoryuken workers. It isn't supported in [Active Job jobs](https://github.com/ruby-shoryuken/shoryuken/wiki/Rails-Integration-Active-Job).

### queue

Default: `default`

The `queue` option associates a queue with a worker.

```ruby
class HelloWorker
  include Shoryuken::Worker
  
  shoryuken_options queue: 'hello'
end
```

You can also pass a block to define the queue:

```ruby
shoryuken_options queue: ->{ "#{ENV['RAILS_ENV']}-hello" }
shoryuken_options queue: ->{ "#{Socket.gethostname}-hello" }
```

Or an array to associate multiple queues to a single worker:

```ruby
shoryuken_options queue: %w[queue1 queue2 queue3]
```

### auto_delete

Default: `false`

When it's enabled, Shoryuken auto deletes messages after their consumption, but only in case, the worker doesn't raise any exception during the consumption.

As `auto_delete` is `false` by default, remember to set it to `true` or call `sqs_msg.delete` before ending the `perform` method, otherwise, the messages will get back to the queue, becoming available to be consumed again.

*Note:* `auto_delete` is `true` by default when using Active Job.

### batch

Default: `false`

When `batch` is `true`, Shoryuken sends the messages in batches instead of individually.

One of the advantages of `batch` is when you use APIs that accept batch, for example [Keen IO](http://keen.io).


```ruby
class KeenIOWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'keen-io', auto_delete: true, batch: true, body_parser: :json

  def perform(sqs_msgs, events)
    Keen.publish_batch('stats' => events)
  end
end
```

If you are using a custom middleware, check [[Middleware]], the `sqs_msg` and `body` are arrays when `batch=true`.

Another important observation regarding batchable workers is if one of your messages causes an exception, consequently `auto_delete` won't be executed for any message.

### body_parser

Default: `:text`

The `body_parser` allows you to parse the body before calling the `perform` method. It accepts: `:json`, `:text`, a block or a class that responds to `.parse`.

```ruby
shoryuken_options body_parser: :json

shoryuken_options body_parser: ->(sqs_msg){ REXML::Document.new(sqs_msg.body) }

shoryuken_options body_parser: JSON
```

### auto_visibility_timeout

Default: `false`

Although I strongly recommend setting the [visibility timeout](http://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html) to a nice long value, sometimes we can't control it. So, for these cases, you can enable the `auto_visibility_timeout` by setting its value to `true`.

When it's enabled, 5 seconds before the default visibility timeout expires, Shoryuken will reset it to its original value again. 

> Be generous while configuring the default visibility_timeout for a queue. If your worker in the worst case takes 2 minutes to consume a message, set the visibility_timeout to at least 4 minutes. It doesn't hurt and it will be better than having the same message being consumed more than the expected.
> http://www.pablocantero.com/blog/2014/11/29/sqs-to-the-rescue/

*Note:* This feature isn't supported when using [`batch=true`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#batch).


### retry_intervals

Default: `nil`

If a worker raises an exception while consuming a message, and does not delete the message, [this message will be available to be consumed again after its visibility timeout expiration](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html).

But if you want to increase or decrease the next time a failing message will be available to be consumed again, you can use `retry_intervals` to implement an [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff).

*Note:* This feature isn't supported when using [`batch=true`](https://github.com/ruby-shoryuken/shoryuken/wiki/Worker-options#batch).

```ruby
shoryuken_options retry_intervals: [300, 1200, 3600] # 5.minutes, 20.minutes and 1.hour
shoryuken_options retry_intervals: ->(attempts) { calculate_next_attempt_interval(attempts) }
```

`attempts` is the message's `ApproximateReceiveCount`. With an array, the first failure uses the first interval, the second failure the second one, and so on. When the attempts outnumber the intervals, Shoryuken uses the last interval. A lambda that returns `nil` (or `false`) does not retry the message with a backoff: it becomes visible again after the queue's visibility timeout.

Keep in mind that Amazon SQS does not officially support exponential backoff, it's something implemented in Shoryuken using the [visibility Timeout](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/AboutVT.html), which can be extended to up to 12 hours. If your interval exceeds the 12 hour maximum, Shoryuken will automatically cap it to the maximum allowed.

> https://docs.aws.amazon.com/AWSSimpleQueueService/latest/APIReference/API_ChangeMessageVisibility.html
>
> You can continue to call ChangeMessageVisibility to extend the visibility timeout to a maximum of 12 hours. If you try to extend beyond 12 hours, the request will be rejected.

### non_retryable_exceptions

Default: `nil`

Available since 7.0.1. When the worker raises one of these exceptions, Shoryuken logs a warning and deletes the message at once, instead of letting it be retried. It accepts an array of exception classes or a lambda that receives the exception and returns `true` for a non-retryable one.

```ruby
class MyWorker
  include Shoryuken::Worker

  shoryuken_options queue: 'my_queue',
                    non_retryable_exceptions: [InvalidInputError, ActiveRecord::RecordNotFound]

  # Or decide per exception
  # shoryuken_options queue: 'my_queue',
  #                   non_retryable_exceptions: ->(error) { error.message.include?('permanent') }

  def perform(sqs_msg, body)
    # ...
  end
end
```

It also works with `batch: true`: all messages of the batch are deleted. It takes precedence over [`retry_intervals`](#retry_intervals).
