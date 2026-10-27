You can tell Shoryuken to load your Rails application by passing the `-R` or `--rails` flag to the `shoryuken` command.

If you load Rails, assuming your workers are located in the `app/jobs` directory, they will be auto-loaded. This means you don't need to require them explicitly with `-r`.

For middleware and other configurations, you might want to create an initializer:

```ruby
# config/initializers/shoryuken.rb

Shoryuken.configure_server do |config|
  # Replace Rails logger so messages are logged wherever Shoryuken is logging
  # Note: this entire block is only run by the processor, so we don't overwrite
  #       the logger when the app is running as usual.

  Rails.logger = Shoryuken::Logging.logger
  Rails.logger.level = Rails.application.config.log_level

  # config.server_middleware do |chain|
  #  chain.add Shoryuken::MyMiddleware
  # end

  # For dynamically adding queues prefixed by Rails.env 
  # Shoryuken.add_group('default', 25)
  # %w(queue1 queue2).each do |name|
  #   Shoryuken.add_queue("#{Rails.env}_#{name}", 1, 'default')
  # end
end
```

*Note:* In the above case, since we are replacing the Rails logger, it's desired that this initializer runs before other initializers (in case they themselves use the logger). Since, by Rails conventions, initializers are executed in alphabetical order, this can be achieved by prepending the initializer filename with `00_` (assuming no other initializers alphabetically precede this one).

### Active Job Support

Yes, Shoryuken supports [Active Job](http://edgeguides.rubyonrails.org/active_job_basics.html)! This means that you can put your jobs in processor-agnostic `ActiveJob::Base` subclasses, and change processors whenever you want (or better yet, switch to Shoryuken from another processor easily!).

It works as expected once your [queuing backend is set](http://edgeguides.rubyonrails.org/active_job_basics.html#setting-the-backend) to `:shoryuken`. Just put your job in `app/jobs`. Here's an example:

```ruby
# app/jobs/process_photo_job.rb
class ProcessPhotoJob < ActiveJob::Base
  queue_as :default

  rescue_from ActiveJob::DeserializationError do |ex|
    Shoryuken.logger.error ex
    Shoryuken.logger.error ex.backtrace.join("\n")
  end

  def perform(photo)
    photo.process_image!
  end
end
```

*Note:* When queueing jobs to be performed in the future (e.g. when setting the `wait` or `wait_until` Active Job options), SQS limits the amount of time to 15 minutes (see [`delay_seconds`](https://docs.aws.amazon.com/sdkforruby/api/Aws/SQS/Client.html#send_message-instance_method)). Shoryuken will raise an exception if you attempt to schedule a job further into the future than this limit.

*Note:* FIFO queues do not support per-message delays. A job with a delay on a FIFO queue raises `Shoryuken::Errors::FifoDelayNotSupportedError`. With `retry_on` on a FIFO queue, set `wait: 0`.


#### Active Job Queue Name Support

*Note:* Active Job allows you to [prefix the queue names](http://edgeguides.rubyonrails.org/active_job_basics.html#queues) of all jobs. Shoryuken supports this behavior natively. By default, though, queue names defined in the config file (or passed to the CLI), are not prefixed similarly. To have Shoryuken honor Active Job prefixes, you must enable that option explicitly. A good place to do that in Rails is in an initializer:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.active_job_queue_name_prefixing = true
```


#### Active Job Queue Adapters

Shoryuken has two queue adapters, the default uses a synchronous producer, that is set up normally in your rails config as:

```ruby
# config/application.rb
config.active_job.queue_adapter = :shoryuken
```

There is also a second option to use an async producer:

```ruby
# config/application.rb
config.active_job.queue_adapter = ActiveJob::QueueAdapters::ShoryukenConcurrentSendAdapter.new
```

If you have a lot of jobs to produce, consider using the `ShoryukenConcurrentSendAdapter`. Another hint for quick job enqueuing for `ActiveRecord` objects is to use a query that only selects the `id`:

```ruby
Rails.application.config.active_job.queue_adapter = ActiveJob::QueueAdapters::ShoryukenConcurrentSendAdapter.new
MyModel.expensive_scope.select(:id).load.each { |obj| MyJob.perform_later(obj) }
```

`ShoryukenConcurrentSendAdapter` sends each job to SQS in the background and returns at once. When Shoryuken stops, it waits for the sends that are still running, so jobs enqueued just before the stop (for example by a worker) reach SQS. This wait and the wait for busy workers together stay within `timeout` (see [[Signals]]). When the time runs out, Shoryuken logs `Timed out draining in-flight ActiveJob sends; some may not have been delivered`.

To wait for the pending sends in your own shutdown sequence, call `wait_for_pending_sends`. It returns `true` when all sends finished, or `false` when the timeout (in seconds, optional) ran out first. A failed send counts as finished; it goes to the adapter's error handler, which by default logs a warning.

```ruby
ActiveJob::Base.queue_adapter.wait_for_pending_sends(5)
```

#### Set SQS SendMessage parameters with `.set`

```ruby
MyJob.set(message_group_id: 'abc123',
          message_deduplication_id: 'dedupe',
          message_attributes: {
            'key' => {
              string_value: 'myvalue',
              data_type: 'String'
            }
          })
     .perform_later(123, 'abc')
```

#### FIFO deduplication

For FIFO queues, the adapter derives a `message_deduplication_id` from the serialized job (without `job_id` and `enqueued_at`). So two enqueues of the same job class with the same arguments within the SQS 5-minute deduplication window collapse into one. To turn this off:

```ruby
# config/initializers/shoryuken.rb
Shoryuken.active_job_fifo_message_deduplication = false
```

Then give a `message_deduplication_id` with `.set`, or enable `ContentBasedDeduplication` on the queue. See [[FIFO Queues]].

#### More features

The adapter also supports bulk enqueuing with `ActiveJob.perform_all_later` ([[Bulk Enqueuing]]), ActiveJob Continuations ([[ActiveJob Continuations]]) and CurrentAttributes persistence ([[CurrentAttributes]]).

### How to use [`retry_on`](https://edgeguides.rubyonrails.org/active_job_basics.html#retrying-or-discarding-failed-jobs)

`retry_on` sends a failed job again as a new message. `attempts` includes the first run, so `attempts: 3` sends the job again 2 times. What happens after the last attempt depends on whether `retry_on` has a block:

- **Without a block**, ActiveJob raises the error again. Shoryuken does not delete the message. With a [redrive policy](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) on the queue, SQS moves the message to the dead-letter queue (DLQ) only after `maxReceiveCount` receives, and each receive runs the job again. Without one, SQS delivers the message again after each visibility timeout.
- **With a block**, ActiveJob calls the block and does not raise the error. The error does not reach Shoryuken, the adapter deletes the message (`auto_delete`), and the message never goes to the DLQ.

To send exhausted jobs to the DLQ, omit the block, or raise the error again in it. Then each receive before the move to the DLQ also calls the block again, so a log line or an alert in the block runs once per receive:

```ruby
class MyJob < ActiveJob::Base
  retry_on StandardError, attempts: 3 do |job, error|
    Rails.logger.error("#{job.class.name} failed: #{error.message}")
    raise error # Shoryuken keeps the message, so the redrive policy can move it to the DLQ
  end

  def perform
    # ...
  end
end
```

On a FIFO queue, the move to the DLQ breaks the order of the message group, see [FIFO Queues](https://github.com/ruby-shoryuken/shoryuken/wiki/FIFO-Queues#failures-and-dead-letter-queues).

#### `executions` and SQS redeliveries

Under Shoryuken, a job's `executions` counts ActiveJob's own runs plus one for each SQS redelivery of the same message. It grows by one on each `retry_on` retry, as in plain ActiveJob. It also grows by one each time SQS delivers the same message again: after an error that `retry_on` does not handle, or after a visibility timeout.

`retry_on` does not count its attempts in `executions`. It keeps one counter per `retry_on` in `exception_executions`, so redeliveries do not use up `retry_on` attempts.

For example, a job with `retry_on RetryableError, attempts: 5` that fails four times and then succeeds:

| Run | Result | `executions` | `retry_on` attempts used before the run |
|-----|--------|--------------|-----------------------------------------|
| 1 | Raises `RetryableError` | 1 | 0 |
| 2 | Raises `RetryableError` | 2 | 1 |
| 3 | Raises an error that `retry_on` does not handle; SQS redelivers the message | 3 | 2 |
| 4 | Raises `RetryableError` | 4 | 2 |
| 5 | Succeeds | 5 | 3 |
